# 🧾 정산 서버 (settlement)

> 이 문서는 담당자만 수정한다. 변경 시 [계약 변경 절차](../service-contracts.md#3-계약-변경-절차)를 따른다.

| 항목 | 값 |
|---|---|
| 담당자 | 허진수 |
| 레포 | [driver-settlement-system](https://github.com/Easy-ADJ/driver-settlement-system) |
| 소유 테이블 | `BATCHES`, `SETTLEMENTS` (+ Spring Batch 메타) |

---

## 정산 배치 (Spring Batch)

매일 자정, **원장에서 미지급금이 있는 기사를 찾아** 수수료 20%를 뺀 지급액을 계산한다.

| 단계 | 내용 |
|---|---|
| 대상 선별 | 원장 `GET /api/ledger/unpaid?date=` 로 미지급 기사 목록을 **한 번에** 조회 |
| 배치풀 | 선별 결과를 메모리상 목록으로 들고 Reader 입력으로 넘긴다 (별도 테이블 없음) |
| ItemReader | 기사별 `GET /api/ledger?driver_id=` — 미지급금 합계와 결제 건별 근거 |
| ItemProcessor | 수수료 20% 차감 → `fare_total` · `fee_amount` · `amount` |
| ItemWriter | `BATCHES`, `SETTLEMENTS` 저장 |
| 지급 분개 | 확정 시 원장 `POST /api/ledger/entries`로 **상쇄 분개를 반드시 남긴다** |
| 청크 | 100건마다 커밋, 실패 시 해당 청크만 롤백 |
| 재실행 안전성 | 같은 `targetDate`가 이미 `CONFIRMED`면 시작 전 예외로 거부 |
| 실행 | 자정 `@Scheduled` + 시연용 수동 실행 API |
| 상태 전이 | `RUNNING` → `CONFIRMED` → `PAID` |

- **수수료율은 하드코딩하지 않는다.** `settlement.fee-rate=0.20`으로 두고 주입받는다.
- 원장이 응답하지 않으면 5xx·타임아웃만 2회 재시도하고, 그래도 실패하면 Job을 실패로 끝낸다. 재실행하면 Spring Batch가 실패 지점부터 재개한다.

> **운임 합계의 출처는 원장이다.** 이전 초안은 결제 서버에서 전일 결제를 읽는 구조였다.
> 지금은 원장이 기사별 결제 합계를 들고 있고, 결제 서버는 대사에서만 부른다.

### 지급 분개를 반드시 남겨야 하는 이유

정산이 확정돼도 원장에 상쇄 분개를 넣지 않으면 **미지급금이 줄지 않는다.**
그러면 다음날 배치가 같은 기사를 다시 선별해 **이미 정산한 금액을 또 정산한다.**

```
8/14 배치 ─▶ 원장 미지급금 42,000원 확인
             │
             ├─▶ SETTLEMENTS 저장 (amount = 33,600)
             └─▶ POST /api/ledger/entries   ← 이게 없으면
                    상쇄 분개 42,000원          8/15 배치가 같은 42,000원을 또 잡는다
             ▼
        미지급금 0원
```

### 정산 내역서

기사별로 "어떤 건이 얼마로 계산됐는지"는 **원장 응답의 결제 건별 내역**으로 답한다. `SETTLEMENTS`에는 확정 시점 금액을 동결해 남긴다.

기사가 문의했을 때 답할 수 있어야 한다는 게 이 프로젝트의 출발점이다.

> ⚠️ **추적성이 원장 응답에 걸려 있다.** `trips`가 사라져 정산 DB에는 결제 단위 근거가 없다.
> `GET /api/ledger?driver_id=` 응답에 **`payment_id`별 금액이 반드시 담겨야** 정산 내역서가 성립한다.

---

## 대사 (Reconciliation)

배치 종료 후 **결제 서버의 전일 결제 합계와 원장 미지급금 합계가 일치하는지** 검증한다.

```
결제 GET /api/payments?date=   ─┐
                                ├─▶ 두 합계 비교
원장 GET /api/ledger/unpaid?date= ─┘
                                     │
               ┌─────────────────────┴──────────────────┐
            일치                                     불일치
               │                                        │
       CONFIRMED로 전이                        CONFIRMED 보류 (배치 중단)
       reconciliation_status = MATCHED         = MISMATCHED
```

- **비교 대상이 중요하다.** 금액을 원장에서 받았으므로 원장을 원장과 비교하면 검증이 아니다. 결제 서버를 독립된 두 번째 출처로 쓴다.
- DB가 나뉘어 FK로 참조 무결성을 보장할 수 없으므로 대사가 그 역할을 대신한다.
- **불일치 시 `CONFIRMED`로 올리지 않고 보류한다.** 틀린 금액이 지급 단계로 넘어가지 않게 한다.
- 알림은 로그 + `BATCHES.reconciliation_status` + 관리자 대시보드 배지까지. SNS·이메일은 제외.

---

## 제공 API

### `GET /api/settlements?driverId=&date=`

관리자용 정산 내역 조회.
⚠️ 현재 이 API를 부르는 클라이언트가 없다. 대시보드는 원장 서버만 보기로 했다.

### `POST /api/settlements/batch?targetDate=`

시연용 배치 수동 실행. 자정을 기다리지 않고 원하는 날짜로 돌린다.
같은 `targetDate`가 이미 `CONFIRMED`면 재실행 거부 규칙이 그대로 막는다.

### `POST /api/settlements/{batchId}/pay`

`CONFIRMED` → `PAID` 전이. **실제 송금은 일어나지 않는다** — 데모 범위에서 `PAID`는 "정산 처리 완료" 표식이다.
대사 불일치로 보류된 배치를 사람이 확인한 뒤 진행시키는 지점이기도 하다.

---

## 다른 서버에 대한 의존

| 대상 | 호출 | 목적 |
|---|---|---|
| 원장 | `GET /api/ledger/unpaid?date=` | 배치 대상 선별 |
| 원장 | `GET /api/ledger?driver_id=` | 금액·근거 조회 |
| 원장 | `POST /api/ledger/entries` | 지급 상쇄 분개 |
| 결제 | `GET /api/payments?date=` | 대사 교차검증 |

**세 서버 중 의존이 가장 많고, 특히 원장에 3개가 걸려 있다.** 원장 API가 확정된 뒤에야 본격 구현이 가능하므로 개발 순서상 마지막에 몰릴 위험이 있다.

로그인 DB에도 직접 접속한다 — 지급에 필요한 계좌번호와 기사 이름 때문이다. `settlement_ro` 계정을 쓰며 DataSource가 2개 필요하다.

---

## 남은 항목

- [ ] `GET /api/ledger/unpaid` · `GET /api/ledger?driver_id=` 응답 스키마 (원장 담당자와 합의)
- [ ] `SETTLEMENTS.ledger_id`를 계속 둘지 — 지급 분개의 ID를 담는 용도로 재정의할지
- [ ] 지급 분개의 `idempotencyKey`를 무엇으로 만들지 (`batchId`+`driverId` 조합 등)
