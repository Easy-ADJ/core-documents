# 👥 팀 역할 분담

**담당자 = 그 서비스의 API와 테이블에 대한 결정권자**이며, 해당 문서를 수정할 수 있는 유일한 사람이다.

| 담당자 | 서비스 | 레포 | 소유 문서 | 소유 테이블 |
|---|---|---|---|---|
| 김주엽 | 결제 (payment) | [driver-payment-system](https://github.com/Easy-ADJ/driver-payment-system) | [services/payment.md](../02-design/services/payment.md) | `PAYMENT_ATTEMPT`, `PAYMENT`, `MONEY` |
| 이치헌 | 원장 (ledger) | [driver-ledger-system](https://github.com/Easy-ADJ/driver-ledger-system) | [services/ledger.md](../02-design/services/ledger.md) | `LEDGER_ENTRIES` |
| 허진수 | 정산 (settlement) | [driver-settlement-system](https://github.com/Easy-ADJ/driver-settlement-system) | [services/settlement.md](../02-design/services/settlement.md) | `BATCHES`, `SETTLEMENTS` |
| 허진수 | 대시보드 (client) | [dashboard-system](https://github.com/Easy-ADJ/dashboard-system) | — | `MANAGER_ACCOUNTS`, `DRIVER_ACCOUNTS` |

## 공동 소유

아래는 한 명이 정할 수 없고, 합의 후 문서를 갱신한다. ([계약 변경 절차](../02-design/service-contracts.md#3-계약-변경-절차))

- [architecture.md](../02-design/architecture.md) — 전체 구성, 서버 간 원자성
- [service-contracts.md](../02-design/service-contracts.md) — 호출 계약, 테이블 접근 규칙
- [erd.md](../02-design/erd.md) — 테이블 구조, 스키마 분리

---

> ⚠️ **원장 서버가 병목이다.**
> 결제와 정산 모두 원장 API에 의존하므로, 원장 API가 확정되기 전에는 나머지 둘이 목(mock)으로만 진행해야 한다.
> 특히 **정산은 원장 API 3개에 의존한다.** 원장이 늦어질수록 정산이 마지막에 몰린다.
