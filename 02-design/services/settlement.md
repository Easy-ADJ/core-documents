# 🧾 정산 서버 (settlement)

> 🚧 **작성 중** — 아래는 [기획서 §3.3·§3.4](../../01-planning/service-spec.md)에서 도출한 초안이다. 구현하면서 담당자가 채운다.
> **이 문서는 담당자만 수정한다.** 변경 시 [계약 변경 절차](../service-contracts.md#3-계약-변경-절차)를 따른다.

| 항목 | 값 |
|---|---|
| 담당자 | 허진수 |
| 레포 | 🚧 미생성 |
| 소유 테이블 | `settlement_batches`, `settlement_items` (+ Spring Batch 메타 테이블) |

---

## 정산 배치 (Spring Batch)

매일 자정, 전날 완료된 결제를 기사별로 묶어 정산 금액(운임의 80%)을 계산한다.

| 단계 | 내용 |
|---|---|
| ItemReader | 결제 서버 `GET /api/payments?date=` 로 전일 결제 내역 조회 — **`payments` 테이블을 직접 읽지 않는다** ([이유](../service-contracts.md#0-테이블-직접-접근-금지)) |
| ItemProcessor | 기사별 집계, 수수료(20%) 차감 |
| ItemWriter | `settlement_batches`, `settlement_items` 저장 |
| 청크 | 일정 건수(예: 100건)마다 커밋, 실패 시 해당 청크만 롤백 |
| 재실행 안전성 | 같은 `targetDate`로 이미 `CONFIRMED`인 배치가 있으면 **시작 전 예외로 거부**. 실패 건은 실패 지점부터 재시작 |
| 상태 전이 | `RUNNING` → `CONFIRMED` → `PAID` |

**정산 내역서**: 기사별로 "어떤 운행 건이 얼마로 계산됐는지" 추적 가능하도록 `settlement_items`에 운행 ID 목록과 금액을 남긴다. 기사가 문의했을 때 답할 수 있어야 한다는 게 이 프로젝트의 출발점이다.

> ⚠️ **서버 분리로 생긴 문제**: Reader가 HTTP 호출이 되면서 결제 서버가 죽어 있으면 배치가 통째로 실패한다. 단일 서버였다면 DB 조회로 끝날 일이었다. 재시도·부분 실패 처리 방식 🚧

---

## 대사 (Reconciliation)

정산 배치 종료 후, **원장상 기사 미지급금 합계와 정산 항목 합계가 일치하는지** 자동 검증한다.

- 원장 서버 `GET /api/ledger/accounts/{accountId}/balance` 호출로 비교
- 불일치 시 🚧 — 로그 / 응답 코드 / 관리자 대시보드 배지. 확장 시 SNS 알림
- 🚧 대사 실패 시 정산을 `CONFIRMED`로 올리지 않고 보류할지

---

## 제공 API

### `GET /api/settlements?driverId=&date=` — 정산 내역 조회

관리자용. 기사별·기간별 정산 내역과 상세 항목을 반환한다.

### 🚧 정산 확정·지급 처리 API

`CONFIRMED` → `PAID` 상태 전이를 API로 노출할지, 배치 내부에서만 처리할지 미확정.

---

## 다른 서버에 대한 의존

| 대상 | 호출 | 목적 |
|---|---|---|
| 결제 서버 | `GET /api/payments?date=` | 배치 Reader — 전일 결제 내역 |
| 원장 서버 | `GET /api/ledger/accounts/{id}/balance` | 대사 검증 |
| 원장 서버 | `POST /api/ledger/entries` | 🚧 정산 확정 시 지급 분개 기록 여부 |

**세 서버 중 의존이 가장 많다.** 결제·원장 API가 확정된 뒤에야 본격 구현이 가능하므로, 개발 순서상 마지막에 몰릴 위험이 있다. 일정에서 이 점을 고려해야 한다 ([schedule.md](../../05-milestones/schedule.md)).

---

## 미확정 항목

- [ ] 배치 스케줄링 방식 — EventBridge vs Spring `@Scheduled` ([architecture.md §5](../architecture.md#5-인프라))
- [ ] 결제 내역 조회 페이지네이션 — 건수가 많을 때 Reader가 어떻게 나눠 읽을지
- [ ] 결제 서버 장애 시 배치 재시도 정책
- [ ] 정산 내역서(CSV/PDF) 생성 및 보관 위치 — S3 vs Supabase Storage
- [ ] 수수료율 20%를 하드코딩할지 설정값으로 뺄지
