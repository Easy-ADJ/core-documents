# 📦 후보 탐색 기록 보관소

프로젝트 주제를 정하는 과정에서 만든 문서들입니다. **현행 기획이 아니므로 개발 기준으로 삼지 마세요.**

현재 유효한 기획은 [service-spec.md](../service-spec.md), 현행 아키텍처는 [02-design/architecture.md](../../02-design/architecture.md)입니다.

## 선정 과정

| 문서 | 내용 |
|---|---|
| [mobility-backend-problems.md](./mobility-backend-problems.md) | 모빌리티 백엔드 실무 문제 1~6 리서치 — 후보의 출발점 |
| [project-candidates.md](./project-candidates.md) | 문제 → 프로젝트 후보 매핑, 비교표, 선정 체크리스트 |

## 미채택 후보 상세 기획

**후보 D(모빌리티 정산 서비스 — EasyADJ)** 로 확정되면서 채택되지 않은 후보들입니다.

| 문서 | 후보 | 대응 문제 |
|---|---|---|
| [candidate-a.md](./candidate-a.md) | A. 실시간 차량 위치 관제 대시보드 | 문제 1. 대규모 동시 위치 업데이트 처리 |
| [candidate-b.md](./candidate-b.md) | B. 배차 매칭 서비스 (이중 배정 방지) | 문제 3. 배차/매칭 시 동시성 제어 |
| [candidate-c.md](./candidate-c.md) | C. IoT 텔레메틱스 수집 파이프라인 | 문제 2. IoT 텔레메틱스 수집 신뢰성 |
| [candidate-e.md](./candidate-e.md) | E. EV 충전기 관제 시스템 (경량 OCPP) | 문제 5. 상태 기반 디바이스 동기화 |
| [candidate-f.md](./candidate-f.md) | F. 실시간 이벤트 알림 시스템 | 문제 6. 대용량 이벤트 기반 알림/상태 전파 |

> 후보 D의 상세 기획은 채택되어 [../service-spec.md](../service-spec.md)로 옮겨졌습니다.
