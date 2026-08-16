# 📒 원장 서버 (ledger)

> 이 문서는 담당자만 수정한다. 결제·정산 **둘 다** 이 API에 의존하므로 변경 파급이 가장 크다.
> 변경 시 [계약 변경 절차](../service-contracts.md#3-계약-변경-절차)를 따른다.

| 항목 | 값 |
|---|---|
| 담당자 | 이치헌 |
| 레포 | [driver-ledger-system](https://github.com/Easy-ADJ/driver-ledger-system) |
| 소유 테이블 | `LEDGER_ENTRIES` |

---

## 제공 API

### `POST /api/ledger/entries` — 분개 기록

하나의 거래에 대해 **차변과 대변을 함께** 기록한다. 결제와 정산이 모두 호출한다.

| | |
|---|---|
| 요청 본문 | `idempotencyKey`, `driverId`, `entryType`, 분개 목록 `[{direction, amount, paymentId}, ...]` |
| 검증 | 차변 합계 ≠ 대변 합계면 `400` + `LEDGER_ENTRY_UNBALANCED`. **애플리케이션에서 검증** |
| 멱등성 | 같은 `idempotencyKey`면 재기록 없이 기존 결과 반환 (`idempotency_key` UNIQUE) |
| 응답 | `201 Created` — 생성된 분개 목록 |

| `entryType` | 상황 | 비고 |
|---|---|---|
| `PAYMENT` | 결제 승인 | 차변(플랫폼 예수금 증가) + 대변(기사 미지급금 증가) |
| `PAYMENT_CANCEL` | 결제 취소 | **역방향 분개를 추가**한다. 원본을 수정·삭제하지 않는다 |
| `SETTLEMENT` | 정산 지급 | 미지급금 상쇄. `paymentId`는 **NULL** |

⚠️ 결제가 **2회까지 재시도**하므로 멱등성이 반드시 동작해야 한다.

### `GET /api/ledger/unpaid?date=` — 미지급 기사 목록

**정산 배치의 시작점.** 해당 일자 기준 미지급금이 남은 기사를 한 번에 반환한다.
기사별로 하나씩 부르면 자정마다 기사 수만큼 호출이 생기기 때문이다.

응답에 최소한 `driverId`와 미지급금 합계가 필요하다.

### `GET /api/ledger?driver_id=` — 기사별 미지급금과 근거

기사 한 명의 **미지급금 합계 + 결제 건별 내역**을 반환한다.
잔액은 **분개의 합으로 계산**하며 잔액 컬럼을 따로 두고 갱신하지 않는다 — 그래야 이상이 생겼을 때 분개 이력을 replay해 추적할 수 있다.

- ⚠️ **응답에 `paymentId`별 금액이 반드시 담겨야 한다.** 정산이 `trips` 부재로 결제 단위 근거를 갖고 있지 않아, **기사 문의에 답하는 유일한 출처가 이 응답**이다.
- 응답이 커지므로 기간 필터(`date` 또는 `from`·`to`)가 함께 필요하다.
- 계정 기반 API(`/accounts/{id}/balance`, `/accounts?ownerType=`)는 **폐기됐다.** `ledger_accounts` 테이블이 없어져 계약이 성립하지 않는다.

### `GET /api/ledger/verify?from=&to=` — 정합성 검증

기간 내 **모든 분개 합이 정확히 0인지** 검증한다. 0이 아니면 정합성 이상이다.

발표 자리에서 "원장이 맞는다"를 즉석에서 보여줄 수 있어 API로 노출한다. 구현은 SUM 쿼리 하나면 된다.
이상은 로그와 응답 코드로 남긴다. 외부 알림은 범위 제외.

---

## 다른 서버에 대한 의존

**다른 서버는 호출하지 않는다.** 원장은 호출만 받는다. 그래서 가장 안정적으로 유지해야 하며, 개발 순서상 원장 API가 먼저 확정돼야 나머지 둘이 진행된다.

단 로그인 DB에는 직접 접속한다 — 분개를 기사 이름과 함께 보여주기 위해서다. `ledger_ro` 계정을 쓰며 DataSource가 2개 필요하다.

---

## 남은 항목

- [ ] `GET /api/ledger/unpaid?date=` 응답 스키마 (정산 담당자와 합의)
- [ ] `GET /api/ledger?driver_id=` 결제 건별 내역 형태
- [ ] 미지급금 계산 기준 — `SETTLEMENT` 상쇄 분개를 뺀 잔액이 맞는지 확인
- [ ] 취소 분개의 멱등 키 — 결제 분개와 같은 값을 쓸 수 없다
- [ ] 잔액 조회 성능 — 분개가 쌓였을 때 매번 SUM하는 비용
