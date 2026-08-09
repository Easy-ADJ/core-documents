# 실시간 이벤트 알림 시스템 — 상세 기획 (후보 F)

> 📦 **보관 문서 — 현행 아님.** 미채택 후보의 기획 초안입니다. 현재 기획은 [service-spec.md](../service-spec.md), 현행 아키텍처는 [architecture.md](../../02-design/architecture.md)를 참조하세요.

> [project-candidates.md](./project-candidates.md)의 **후보 F(실시간 이벤트 알림 시스템)** — 문제 6 "대용량 이벤트 기반 알림/상태 전파" 대응 — 를 실제 개발에 들어갈 수 있는 수준으로 구체화한 문서다. 기술 스택은 Java 17+ / **Spring Boot 4** / **AWS** 기준.
>
> ⚠️ [project-candidates.md](./project-candidates.md)에서는 단독 프로젝트보다 후보 A/B/D의 확장 기능으로 결합하는 것을 권장했다. 이 문서는 **단독으로 선택할 경우**를 가정해 구체화했으며, 다른 후보와 결합할 때는 "이벤트 발행자"만 해당 후보의 도메인 이벤트(배차 완료, 결제 완료 등)로 교체하면 그대로 재사용할 수 있다.

---

## 1. 문제 재정의

"배차 완료", "차량 도착", "결제 완료" 같은 이벤트를 다수의 클라이언트(승객 앱, 기사 앱, 관제 대시보드)에 실시간으로, 유실 없이 전파해야 한다.

- 폴링 방식은 서버 부하가 크고 지연이 발생하며, 단순 WebSocket 브로드캐스트만으로는 서버 재시작·장애 시 이벤트가 유실될 수 있다.
- 하나의 이벤트가 여러 독립적인 소비자(알림 발송, 통계 집계, 관제 대시보드 갱신)에게 동시에 필요한 경우, 발행자가 각 소비자를 직접 호출하는 대신 **SNS(팬아웃) + SQS(큐잉)** 조합으로 디커플링하는 것이 표준 패턴이다. ([Building Fan-Out Serverless Architectures Using SNS, SQS and Lambda](https://medium.com/@ayushjudesharp/building-fan-out-serverless-architectures-using-sns-sqs-and-lambda-event-driven-architecture-911a7b4eadfb))
- SQS는 소비자가 메시지를 삭제하기 전까지 visibility timeout 동안 다른 소비자에게 보이지 않게 하여 유실 없는 처리를 보장하고, 반복 실패한 메시지는 Dead-Letter Queue(DLQ)로 격리해 무한 재시도를 막는다. ([AWS SQS and SNS: Messaging and Event-Driven Architecture](https://techoral.com/aws/aws-sqs-sns-messaging.html))

---

## 2. 시나리오

> 당신은 모빌리티 플랫폼 "D-Move"의 백엔드 개발팀에 합류했다.
>
> 배차가 완료되면 승객 앱에 "기사님이 배정되었습니다" 알림이 가야 하고, 동시에 관제 대시보드의 콜 현황판도 갱신되어야 하며, 통계 시스템에도 이벤트가 쌓여야 한다.
>
> 초기 구현은 배차 서비스 코드 안에서 "알림 보내기", "대시보드 갱신하기", "통계 적재하기"를 순서대로 직접 호출하는 방식이었는데, 문제가 생겼다.
> - 알림 발송 서버가 잠깐 느려지자 배차 API 자체의 응답 시간이 함께 늘어졌다.
> - 서버가 재배포되며 잠깐 재시작하는 동안 발생한 배차 완료 이벤트 몇 건이 그대로 사라져, 일부 승객이 배정 알림을 받지 못했다.
>
> 팀 리드가 말했다.
> "배차 로직과 '이 이벤트를 누구에게 어떻게 전달할지'는 분리되어야 해요. 그리고 서버가 재시작해도 이벤트는 유실되면 안 됩니다."

---

## 3. 핵심 기능 명세

| 기능 | 설명 |
|---|---|
| 이벤트 발행 API(내부) | 도메인 이벤트 발생 시(예: 배차 완료) `EventPublisher`가 **Amazon SNS 토픽**에 이벤트 발행 — 배차 로직과 알림 전달 로직을 분리 |
| 팬아웃 구독 | SNS 토픽에 여러 **Amazon SQS 큐**(알림용/대시보드 갱신용/통계 적재용)를 구독시켜, 하나의 이벤트를 독립적인 소비자들이 각자 처리 |
| 소비자(Consumer) | Spring Boot 4 애플리케이션이 각 SQS 큐를 폴링해 이벤트 소비 — 예: 알림 큐 소비자는 WebSocket으로 승객에게 push |
| WebSocket/SSE 구독 서버 | 승객 앱이 WebSocket으로 연결해두면, 알림 큐 소비자가 처리한 이벤트를 해당 세션으로 실시간 전달 |
| 재연결 시 유실 이벤트 재전송 | 클라이언트가 재연결할 때 마지막으로 받은 이벤트 시각(또는 시퀀스)을 서버에 전달하면, 그 이후 발생한 이벤트를 DB에서 조회해 재전송 |
| Dead-Letter Queue | 반복 처리 실패(예: WebSocket 연결이 없는데 계속 전달 시도)한 메시지는 지정 횟수 이후 DLQ로 이동, 별도 모니터링 |
| 이벤트 조회 API | `GET /api/events?since=` — 특정 시점 이후 발생한 이벤트 이력 조회(재전송/디버깅용) |

---

## 4. 아키텍처

### 4.1 시스템 구성도 (텍스트)

```
[배차/결제 등 도메인 서비스] ──이벤트 발행──▶ [Amazon SNS Topic]
                                                     │  (팬아웃)
                              ┌──────────────────────┼──────────────────────┐
                              ▼                       ▼                      ▼
                     [SQS: 알림 큐]           [SQS: 대시보드 갱신 큐]   [SQS: 통계 적재 큐]
                              │                       │                      │
                              ▼                       ▼                      ▼
                [Spring Boot Consumer]      [Spring Boot Consumer]   [Spring Boot Consumer]
                - WebSocket push            - 대시보드 캐시 갱신        - 통계 DB 적재
                              │
                              ▼
                [승객/기사 앱 (WebSocket 세션)]

  (각 큐에는 DLQ가 연결되어, 반복 실패 메시지는 격리되어 별도 모니터링)
```

### 4.2 데이터 모델 개요

- **DomainEvent**(이벤트 이력, RDS) — id, eventType(DISPATCH_COMPLETED/PAYMENT_COMPLETED 등), payload(JSON), occurredAt — 재전송·디버깅을 위한 이벤트 소싱 성격의 저장소
- **NotificationDeliveryLog**(알림 전달 이력) — id, eventId, targetUserId, channel(WEBSOCKET), status(DELIVERED/FAILED/QUEUED_FOR_RETRY)

### 4.3 기술 선택과 근거

| 요소 | 선택 | 근거 |
|---|---|---|
| 팬아웃 | Amazon SNS → 다중 SQS 구독 | 발행자는 SNS 토픽 하나에만 발행하면 되고, 새로운 소비자가 추가되어도 발행자 코드를 수정할 필요 없음 |
| 유실 방지 | SQS visibility timeout + 명시적 삭제 | 소비자가 처리를 완료(삭제)하기 전까지 메시지가 큐에 남아있어, 소비자 장애 시에도 다른 인스턴스가 재처리 가능 |
| 실패 격리 | Dead-Letter Queue(DLQ) | 반복 실패하는 메시지가 큐를 막지 않도록 별도 격리, 운영팀이 사후 확인 |
| 실시간 전달 | Spring WebSocket | SQS 컨슈머가 처리한 이벤트를 최종적으로 클라이언트에 push하는 계층 |
| 이벤트 재전송 | RDS에 이벤트 이력 별도 저장 | WebSocket 연결이 끊겨 있던 동안의 이벤트를 재연결 시 조회해 재전송 가능하게 함 |

- 참고: [SNS + SQS => Fanout](https://medium.com/asktechjedi/sns-sqs-fanout-5eb072457362)
- 참고: [Mastering the SNS + SQS Fan-Out Pattern in Event-Driven Systems](https://nemanjatanaskovic.com/mastering-the-sns-sqs-fan-out-pattern-in-event-driven-systems-2/)
- 참고: [Send Fanout Event Notifications - AWS 공식 핸즈온](https://aws.amazon.com/getting-started/hands-on/send-fanout-event-notifications/)

### 4.4 신뢰성 전략

- SQS 큐마다 재시도 횟수(`maxReceiveCount`)를 설정하고, 초과 시 DLQ로 이동하도록 구성
- WebSocket 연결이 없는 사용자에게는 즉시 실패 처리하지 않고, 이벤트 이력을 남겨두었다가 **재연결 시점에 미수신 이벤트를 일괄 재전송**하는 방식으로 처리(오프라인 상태에서도 이벤트가 사라지지 않도록 함)
- visibility timeout은 평균 처리 시간보다 넉넉히 설정해, 처리 중인 메시지를 다른 소비자가 중복 처리하지 않도록 함

---

## 5. 개발 로드맵 (실제 개발 가능 기간 3~4주 기준)

> 프로그램 일정 기준: 6주차(07.27~07.31) 기획 마무리 → **7주차(08.03~08.07) / 8주차(08.10~08.14) / 9주차(08.17~08.21)** 3주간 개발 → 9주차 팀밋업(08.18) 중간 점검 → 10주차 성과교류회(08.25) 최종 발표.

### 7주차 (08.03~08.07) — 이벤트 발행 및 팬아웃 구조

- Day 1~2: `DomainEvent` 모델 설계, 더미 도메인 이벤트(예: 가상의 "배차 완료") 발행 API 작성
- Day 3~4: SNS 토픽 생성, 알림/대시보드/통계용 SQS 큐 3개 구독 연결
- Day 5: 이벤트 발행 시 3개 큐 모두에 메시지가 도착하는지 확인
- **완료 기준**: 이벤트 하나를 발행하면 3개의 독립된 큐에서 각각 메시지를 수신함을 확인

### 8주차 (08.10~08.14) — 소비자 구현 및 WebSocket 전달

- Day 1~2: 알림 큐 소비자 구현(Spring Boot SQS Listener), 처리 완료 시 메시지 삭제
- Day 3~4: WebSocket 서버 구현, 소비자가 처리한 이벤트를 대상 사용자 세션에 push
- Day 5: DLQ 설정 및 강제로 실패를 유발해 DLQ로 이동하는 것을 확인
- **완료 기준**: 발행된 이벤트가 WebSocket을 통해 실시간으로 클라이언트에 도달, 반복 실패 메시지는 DLQ에서 확인 가능

### 9주차 (08.17~08.21) — 재전송, 통합, 발표 준비

- Day 1~2: 재연결 시 유실 이벤트 재전송 로직 구현(`GET /api/events?since=` 기반)
- Day 3: 대시보드/통계 소비자까지 포함한 전체 파이프라인 통합 테스트
- Day 4: AWS 배포(SNS/SQS 구성 확정), 서버 재시작 시나리오로 유실 없음을 검증
- Day 5(팀밋업 08.18 전후): "서버 재시작 중 발생한 이벤트도 재연결 시 모두 수신됨"을 보여주는 데모 시나리오 및 발표자료 준비
- **완료 기준**: 서버를 의도적으로 재시작해도 클라이언트가 재연결 후 놓친 이벤트를 전부 수신함을 라이브로 시연

---

## 6. 참고 자료

- [Building Fan-Out Serverless Architectures Using SNS, SQS and Lambda](https://medium.com/@ayushjudesharp/building-fan-out-serverless-architectures-using-sns-sqs-and-lambda-event-driven-architecture-911a7b4eadfb)
- [SNS + SQS => Fanout](https://medium.com/asktechjedi/sns-sqs-fanout-5eb072457362)
- [Mastering the SNS + SQS Fan-Out Pattern in Event-Driven Systems](https://nemanjatanaskovic.com/mastering-the-sns-sqs-fan-out-pattern-in-event-driven-systems-2/)
- [Send Fanout Event Notifications - AWS 공식 핸즈온](https://aws.amazon.com/getting-started/hands-on/send-fanout-event-notifications/)
- [AWS SQS and SNS: Messaging and Event-Driven Architecture](https://techoral.com/aws/aws-sqs-sns-messaging.html)
- [Designing Uber: Geospatial Indexing, WebSockets, and Distributed Locks](https://dev.to/ganesh_parella/designing-uber-geospatial-indexing-websockets-and-distributed-locks-4mhb)
- [The Power of MQTT and Confluent in Fleet Management](https://www.confluent.io/blog/fleet-management/)
