# 💳 결제 서버 (payment)

> 이 문서는 담당자만 수정한다. 다른 서버가 이 API에 의존하므로 변경 시 [계약 변경 절차](../service-contracts.md#3-계약-변경-절차)를 따른다.

| 항목 | 값 |
|---|---|
| 담당자 | 김주엽 |
| 레포 | [driver-payment-system](https://github.com/Easy-ADJ/driver-payment-system) |
| 소유 테이블 | `PAYMENT_ATTEMPT`, `PAYMENT`, `MONEY` |

---

## 제공 API

### `POST /api/payments` — 결제 요청

운행 종료 시 결제를 청구한다. 이 서비스의 핵심이자 **멱등성이 걸린 지점**이다.

| | |
|---|---|
| 요청 헤더 | `Idempotency-Key: <UUID>` **필수** (클라이언트 생성) |
| 요청 본문 | `driverId`, `amount` |
| 응답 | `201 Created` — `paymentId`, `status`, `amount` |
| 재요청 | 같은 키로 오면 재실행 없이 첫 처리 결과를 반환 |
| 동시성 | `PAYMENT.idempotency_key` UNIQUE로 동시 요청 중 하나만 성공 |

**금액의 흐름** — 클라이언트가 요청에 담아 보낸 값으로 결제를 준비하고, **카카오페이 승인 응답 금액으로 `MONEY`를 확정**한다. 이후 원장·정산이 보는 것은 항상 `MONEY`의 확정 금액이다.

⚠️ 서버가 금액 원본을 갖고 있지 않아 **위변조를 검증할 수 없다.** `trips` 테이블이 ERD에서 빠지면서 생긴 제약이며 데모 범위에서 감수한다.

### 처리 순서

```
1. Idempotency-Key로 기존 결제 확인 → 있으면 첫 결과 반환하고 종료
2. PAYMENT · PAYMENT_ATTEMPT 저장
3. 카카오페이 승인 → MONEY 확정 저장
4. 원장 POST /api/ledger/entries 호출
     ├─ 성공 ─────────▶ 결제 완료
     ├─ 5xx·타임아웃 ─▶ 2회 재시도 → 실패 시 결제도 롤백
     └─ 4xx ──────────▶ 재시도 없이 결제 실패
```

원장에 빠진 결제건이 생기면 정산 금액이 곧바로 틀리므로 **결제 가용성보다 금액 정합성을 택했다.** ([architecture.md §3](../architecture.md#3-결제와-원장의-원자성))
재시도가 실제로 일어나므로 원장에 보내는 `idempotencyKey`는 **재시도 간에 동일해야 한다.**

### `GET /api/payments/{paymentId}` — 결제 상태 조회

`SUCCESS` / `FAILED` / `CANCELED` 확인.

### `POST /api/payments/{paymentId}/cancel` — 취소·환불

원장에 **상쇄(역방향) 분개를 추가 기록**한다. 원본 분개를 삭제하지 않는다.

### `GET /api/payments?date=yyyy-MM-dd` — 일자별 결제 내역

**정산 서버 전용 — 대사 교차검증용이다.**

⚠️ **정산 배치의 ItemReader가 아니다.** 운임 합계의 출처가 원장으로 바뀌면서, 이 API는 대사 단계에서 **원장 합계와 비교할 독립된 두 번째 출처**로만 쓰인다.

페이지네이션은 `page` / `size` 오프셋 방식으로 제공한다.

---

## 다른 서버에 대한 의존

| 대상 | 호출 | 목적 |
|---|---|---|
| 원장 | `POST /api/ledger/entries` | 결제 승인·취소 분개 기록 |

로그인 DB에도 직접 접속한다 — `PAYMENT.driver_id`가 실재하는 기사인지 확인하기 위해서다. `payment_ro` 계정을 쓰며 DataSource가 2개 필요하다.

---

## 남은 항목

- [ ] 원장에 보낼 `idempotencyKey`를 결제의 `Idempotency-Key`와 같은 값으로 쓸지, 파생시킬지 (취소 분개는 다른 키가 필요하다)
- [ ] `Idempotency-Key` 유효 기간 (영구 보관 vs N일 후 정리)
- [ ] `GET /api/payments?date=`가 `SUCCESS`만 반환할지
- [ ] 카카오페이 연동 범위 — 실제 승인까지 갈지 모의 처리로 갈지
