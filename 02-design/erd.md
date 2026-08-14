# 📐 ERD 및 테이블 소유권

> 🚧 <u>**작성 중**</u> — ERDCloud 다이어그램 확정 후 채운다.<br/>
> 아래 테이블 목록은 [기획서 §4.2](../01-planning/service-spec.md)의 데이터 모델 개요를 서비스별로 나눈 것이다.<br/>
> 다이어그램: [Team ERDCloud](https://www.erdcloud.com/team/xyiBeZRo9gEJPk2WZ)

---

## 1. DB 구성

<u>**시스템마다 DB 인스턴스를 하나씩 둔다.**</u> 각 서버는 자기 DB에만 접속한다.

| DB | 소유 서비스 | 담당자 | 담긴 테이블 |
|---|---|---|---|
| DB #1 | 결제 (payment) | 김주엽 | `trips`, `payments` |
| DB #2 | 원장 (ledger) | 이치헌 | `ledger_accounts`, `ledger_entries` |
| DB #3 | 정산 (settlement) | 허진수 | `settlement_batches`, `settlement_items` (+ Spring Batch 메타) |

DB를 나눴으므로 <u>**테이블 소유권이 DB 경계로 강제된다.**</u> 남의 테이블은 조인하고 싶어도 접속 자체가 불가능하다. 경계를 넘는 데이터는 전부 API를 거친다. ([접근 규칙](./service-contracts.md#0-테이블-직접-접근-금지))

- 🚧 DB 제품은 미확정 — **AWS Aurora** 혹은 **Supabase**. ([architecture.md §5](./architecture.md#5-인프라))
- 접속 정보와 연결 설정: [dev-environment.md](../03-development/dev-environment.md)

---

## 2. 테이블 소유권

<u>**소유자 = 그 테이블의 스키마를 변경할 수 있는 유일한 사람.**</u><br/>
DB가 나뉘어 물리적 사고는 막히지만, 스키마 변경은 <u>**그 테이블을 읽는 상대 서비스의 API 응답을 바꾼다.**</u> 그래서 소유권 합의는 여전히 필요하다.

| 테이블 | 소유 서비스 | 담당자 | 용도 |
|---|---|---|---|
| `trips` | 결제 | 김주엽 | 운행 — `riderId`, `driverId`, `fare`, `completedAt` |
| `payments` | 결제 | 김주엽 | 결제 — `tripId`, `idempotencyKey`(UNIQUE), `amount`, `status`, `requestedAt` |
| `ledger_accounts` | 원장 | 이치헌 | 계정 — `ownerType`(PLATFORM/DRIVER), `ownerId` |
| `ledger_entries` | 원장 | 이치헌 | 분개 — `transactionId`, `accountId`, `direction`(DEBIT/CREDIT), `amount`, `createdAt` |
| `settlement_batches` | 정산 | 허진수 | 배치 실행 이력 — `targetDate`, `status`, `executedAt` |
| `settlement_items` | 정산 | 허진수 | 정산 상세 — `settlementBatchId`, `driverId`, `tripIds`, `amount`, `payoutStatus` |

> Spring Batch를 쓰면 `BATCH_JOB_INSTANCE` 등 메타 테이블이 자동 생성된다. 정산 DB에 함께 둔다.

---

## 3. 주요 제약

| 테이블 | 제약 | 이유 |
|---|---|---|
| `payments` | `idempotency_key` <u>**UNIQUE**</u> | 동시 중복 결제 요청 중 하나만 성공시키는 핵심 장치 ([기획서 §4.3](../01-planning/service-spec.md)) |
| `ledger_entries` | `transaction_id` 단위로 차변 합 = 대변 합 | 이중기입 정합성. DB 제약으로 강제할지 애플리케이션에서 검증할지 🚧 |
| `ledger_entries` | 수정·삭제 금지 (append-only) | 취소는 삭제가 아니라 <u>**상쇄 분개 추가**</u>로 처리 |
| `settlement_batches` | `target_date` 단위 중복 CONFIRMED 방지 | 정산 배치 재실행 시 중복 집계 차단 |

> <u>**핵심 설계 원칙**</u>(기획서 §4.2): 잔액을 직접 갱신하지 않고 <u>**분개의 합으로 항상 재계산한다.**</u><br/>
> 그래야 잔액이 이상할 때 분개 이력을 replay해 원인을 추적할 수 있다.

---

## 4. 서비스 경계를 넘는 참조

DB가 나뉘어 <u>**경계를 넘는 FK는 걸 수 없다.**</u> 상대 서비스의 식별자는 FK 없이 값으로만 보관한다.

| 보관하는 곳 | 값 | 원본 소유 |
|---|---|---|
| `payments.trip_id` | 운행 ID | 결제 (같은 DB, FK 가능) |
| `settlement_items.trip_ids` | 정산에 포함된 운행 ID 목록 | 결제 DB — <u>**FK 불가**</u> |
| `ledger_entries.transaction_id` | 결제·정산이 발급한 거래 키 | 호출자 — <u>**FK 불가**</u> |

- DB가 참조 무결성을 보장해주지 않으므로, 존재하지 않는 ID가 섞여도 insert가 성공한다.
- 이 불일치를 잡아내는 것이 정산의 <u>**대사(reconciliation)**</u> 다. ([settlement.md](./services/settlement.md))

---

## 5. 채워야 할 것

- [ ] ERDCloud 다이어그램 확정 후 이미지 첨부 (`images/` 폴더에 저장)
- [ ] 각 테이블의 전체 컬럼·타입·nullable·기본값
- [ ] DB 내부 FK 관계 (경계를 넘지 않는 것)
- [ ] 인덱스 (정산 배치의 `payments` 날짜 범위 조회, 원장 잔액 계산의 `account_id` 집계)
- [ ] 차변 합 = 대변 합 검증을 DB 제약으로 걸지 애플리케이션에서 할지 결정

---

## 관련 문서

- 전체 구성: [Architecture 문서](./architecture.md)
- 접근 규칙: [Service Contracts 문서](./service-contracts.md)
- 데이터 모델 배경: [확정 기획서 §4.2](../01-planning/service-spec.md)
