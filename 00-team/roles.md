# 👥 팀 역할 분담

서비스 3개를 담당자별로 나눠 개발한다. **담당자 = 그 서비스의 API와 테이블에 대한 결정권자**이며, 해당 문서를 수정할 수 있는 유일한 사람이다.

| 담당자 | 서비스 | 레포 | 소유 문서 | 소유 테이블 |
|---|---|---|---|---|
| 김주엽 | 결제 (payment) | 🚧 미생성 | [services/payment.md](../02-design/services/payment.md) | `trips`, `payments` |
| 이치헌 | 원장 (ledger) | 🚧 미생성 | [services/ledger.md](../02-design/services/ledger.md) | `ledger_accounts`, `ledger_entries` |
| 허진수 | 정산 (settlement) | [driver-settlement-system](https://github.com/Easy-ADJ/driver-settlement-system) | [services/settlement.md](../02-design/services/settlement.md) | `settlement_batches`, `settlement_items` |

## 공동 소유

아래는 한 명이 정할 수 없다. 합의 후 갱신한다 ([계약 변경 절차](../02-design/service-contracts.md#3-계약-변경-절차)).

- [architecture.md](../02-design/architecture.md) — 전체 구성, 서버 간 원자성 처리 방식
- [service-contracts.md](../02-design/service-contracts.md) — 서버 간 호출 계약, 테이블 접근 규칙
- [erd.md](../02-design/erd.md) — 테이블 구조, 스키마 분리 방식

## AI 정산 웹/앱 사이트 디자인·개발

Stitch, Figma, CLAUDE, Codex

---

> **원장 서버가 병목이다.** 결제와 정산 **둘 다** 원장 API에 의존하므로([호출 관계](../02-design/service-contracts.md#1-호출-관계)), 원장 API가 확정되기 전에는 나머지 둘이 목(mock)으로만 진행할 수 있다. 원장 API 시그니처를 가장 먼저 고정하는 편이 전체 일정에 유리하다.
