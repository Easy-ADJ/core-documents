# 📐 ERD 및 테이블 소유권

> 🚧 **작성 중** — ERDCloud 다이어그램 확정 후 채운다. 아래 테이블 목록은 [기획서 §4.2](../01-planning/service-spec.md)의 데이터 모델 개요를 서비스별로 나눈 것이다.
> 다이어그램: [Team ERDCloud](https://www.erdcloud.com/team/xyiBeZRo9gEJPk2WZ)

---

## 1. DB 구성

**Supabase PostgreSQL 인스턴스 1개를 3개 서비스가 공유한다.**

서비스는 나뉘었지만 DB는 하나다. 따라서 "어느 테이블이 누구 것인가"가 코드가 아니라 **합의로만 정해진다.** 아래 소유권 표가 그 합의다.

- 접속 정보와 연결 설정: [dev-environment.md](../03-development/dev-environment.md)
- 다른 서비스 테이블에 직접 접근하지 않는 규칙: [service-contracts.md §0](./service-contracts.md#0-테이블-직접-접근-금지)

---

## 2. 테이블 소유권

**소유자 = 그 테이블의 스키마를 변경할 수 있는 유일한 사람.** 다른 담당자는 소유자를 거치지 않고 컬럼을 추가하거나 타입을 바꾸지 않는다.

| 테이블 | 소유 서비스 | 담당자 | 용도 |
|---|---|---|---|
| `trips` | 결제 | 김주엽 | 운행 — `riderId`, `driverId`, `fare`, `completedAt` |
| `payments` | 결제 | 김주엽 | 결제 — `tripId`, `idempotencyKey`(UNIQUE), `amount`, `status`, `requestedAt` |
| `ledger_accounts` | 원장 | 이치헌 | 계정 — `ownerType`(PLATFORM/DRIVER), `ownerId` |
| `ledger_entries` | 원장 | 이치헌 | 분개 — `transactionId`, `accountId`, `direction`(DEBIT/CREDIT), `amount`, `createdAt` |
| `settlement_batches` | 정산 | 허진수 | 배치 실행 이력 — `targetDate`, `status`, `executedAt` |
| `settlement_items` | 정산 | 허진수 | 정산 상세 — `settlementBatchId`, `driverId`, `tripIds`, `amount`, `payoutStatus` |

> Spring Batch를 쓰면 `BATCH_JOB_INSTANCE` 등 메타 테이블이 자동 생성된다. 정산 서비스 소유로 본다.

---

## 3. 소유권을 무엇으로 표현할까

> 🚧 **미확정 — 8/11 회의 안건.**

2절의 소유권은 지금 **표에 적힌 약속일 뿐**이다. 실수로 남의 테이블을 건드려도 아무 일도 일어나지 않는다. 이를 DB 수준에서 강제할지 정해야 한다.

| | (A) PostgreSQL 스키마 분리 | (B) `public` 단일 스키마 + 명명 규칙 |
|---|---|---|
| 구성 | `payment.`, `ledger.`, `settlement.` 3개 스키마 | 전부 `public`, 테이블명 접두사로 구분 |
| 경계 강제 | **DB 권한(GRANT)으로 실제 차단 가능** | 불가 — 규칙에만 의존 |
| 설정 부담 | 서비스별 DB 롤·권한 설정 필요 | 없음 |
| 마이그레이션 | 스키마별로 분리돼 충돌이 줄어듦 | 한 스키마에 6개 테이블이 섞임 |
| Supabase | 대시보드에서 스키마 전환 필요 | 기본 뷰에서 바로 보임 |

(A)를 택하면 [service-contracts.md §0](./service-contracts.md#0-테이블-직접-접근-금지)의 규칙이 문서가 아니라 **DB 권한으로 지켜진다.** 초기 설정에 반나절 정도 들지만, 3주 내내 "실수로 남의 테이블 조인" 위험이 사라진다.

---

## 4. 주요 제약

| 테이블 | 제약 | 이유 |
|---|---|---|
| `payments` | `idempotency_key` **UNIQUE** | 동시 중복 결제 요청 중 하나만 성공시키는 핵심 장치 ([기획서 §4.3](../01-planning/service-spec.md)) |
| `ledger_entries` | `transaction_id` 단위로 차변 합 = 대변 합 | 이중기입 정합성. DB 제약으로 강제할지 애플리케이션에서 검증할지 🚧 |
| `ledger_entries` | 수정·삭제 금지 (append-only) | 취소는 삭제가 아니라 **상쇄 분개 추가**로 처리 |
| `settlement_batches` | `target_date` 단위 중복 CONFIRMED 방지 | 정산 배치 재실행 시 중복 집계 차단 |

> **핵심 설계 원칙**(기획서 §4.2): 잔액을 직접 갱신하지 않고 **분개의 합으로 항상 재계산한다.** 그래야 잔액이 이상할 때 분개 이력을 replay해 원인을 추적할 수 있다.

---

## 5. 채워야 할 것

- [ ] ERDCloud 다이어그램 확정 후 이미지 첨부 (`images/` 폴더에 저장)
- [ ] 각 테이블의 전체 컬럼·타입·nullable·기본값
- [ ] 테이블 간 FK 관계 — **특히 서비스 경계를 넘는 FK를 둘지** (예: `settlement_items.trip_ids`가 `trips`를 FK로 참조하면 정산이 결제 테이블에 물리적으로 묶인다)
- [ ] 인덱스 (정산 배치의 `payments` 날짜 범위 조회, 원장 잔액 계산의 `account_id` 집계)
- [ ] 3절 논점 결정 후 최종 스키마 구조 반영

---

## 관련 문서

- 전체 구성: [architecture.md](./architecture.md)
- 접근 규칙: [service-contracts.md](./service-contracts.md)
- 데이터 모델 배경: [확정 기획서 §4.2](../01-planning/service-spec.md)
