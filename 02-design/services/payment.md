# 💳 결제 서버 (payment)

> 🚧 **작성 중** — 아래는 [기획서 §3.1](../../01-planning/service-spec.md)에서 도출한 초안이다. 구현하면서 담당자가 채운다.
> **이 문서는 담당자만 수정한다.** 다른 서버가 이 API에 의존하므로, 변경 시 [계약 변경 절차](../service-contracts.md#3-계약-변경-절차)를 따른다.

| 항목 | 값 |
|---|---|
| 담당자 | 김주엽 |
| 레포 | 🚧 미생성 |
| 소유 테이블 | `trips`, `payments` |

---

## 제공 API

### `POST /api/payments` — 결제 요청

운행(Trip) 종료 시 결제를 청구한다. 이 서비스의 핵심이자 **멱등성이 걸린 지점**이다.

| | |
|---|---|
| 요청 헤더 | `Idempotency-Key: <UUID>` **필수** (클라이언트 생성) |
| 요청 본문 | `tripId`, `amount` |
| 응답 | `201 Created` — `paymentId`, `status`, `amount` |
| 재요청 | 같은 `Idempotency-Key`로 다시 오면 **결제를 재실행하지 않고 첫 처리 결과를 그대로 반환** |
| 동시성 | `payments.idempotency_key` UNIQUE 제약으로 동시 요청 중 하나만 성공 |

처리 순서 🚧 — [architecture.md §3](../architecture.md#3-결제와-원장의-원자성)의 결정에 따라 달라진다. (B) 선택 시 `Payment` 저장 후 원장 서버 `POST /api/ledger/entries` 호출, 실패 처리 방식은 미확정.

### `GET /api/payments/{paymentId}` — 결제 상태 조회

`SUCCESS` / `FAILED` / `CANCELED` 상태 확인.

### `POST /api/payments/{paymentId}/cancel` — 결제 취소·환불

취소 시 원장에 **상쇄(역방향) 분개를 추가 기록**한다. 원본 분개를 삭제하지 않는다.

### `GET /api/payments?date=yyyy-MM-dd` — 일자별 결제 내역 조회

**정산 서버 전용.** 정산 배치의 ItemReader가 전일 완료 결제를 읽어간다.

- 🚧 페이지네이션 필요 여부 — 건수가 많으면 `page`/`size` 또는 커서 방식
- 🚧 `status=SUCCESS`만 반환할지, 전체 반환 후 정산이 거를지

---

## 다른 서버에 대한 의존

| 대상 | 호출 | 목적 |
|---|---|---|
| 원장 서버 | `POST /api/ledger/entries` | 결제 승인·취소 시 분개 기록 |

전체 호출 관계는 [service-contracts.md §1](../service-contracts.md#1-호출-관계) 참조.

---

## 미확정 항목

- [ ] 원장 호출 실패 시 처리 방식 ([architecture.md §3](../architecture.md#3-결제와-원장의-원자성))
- [ ] `Idempotency-Key` 유효 기간 (영구 보관 vs N일 후 정리)
- [ ] 결제 요청 본문에 `amount`를 클라이언트가 보낼지, `tripId`로 서버가 조회할지 — 후자가 위변조에 안전
- [ ] 외부 PG 연동 여부 (기획서는 언급 없음, 3주 일정상 모의 처리로 갈 가능성)
- [ ] `trips` 데이터를 어떻게 채울지 (별도 API vs 시드 데이터)
