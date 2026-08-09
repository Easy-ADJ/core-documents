# 배차 매칭 서비스 (이중 배정 방지) — 상세 기획 (후보 B)

> 📦 **보관 문서 — 현행 아님.** 미채택 후보의 기획 초안입니다. 현재 기획은 [service-spec.md](../service-spec.md), 현행 아키텍처는 [architecture.md](../../02-design/architecture.md)를 참조하세요.

> [project-candidates.md](./project-candidates.md)의 **후보 B(배차 매칭 서비스, 이중 배정 방지)** — 문제 3 "배차/매칭 시 동시성 제어" 대응 — 를 실제 개발에 들어갈 수 있는 수준으로 구체화한 문서다. 기술 스택은 Java 17+ / **Spring Boot 4** / **AWS** 기준.

---

## 1. 문제 재정의

승객(또는 화물) 호출이 발생하면 주변 여러 기사에게 동시에 알림이 가고, 그중 가장 먼저 응답한 한 명에게만 배정되어야 한다. 분산 환경에서 "정확히 한 번만 배정된다"는 정합성을 보장하는 것이 핵심 과제다.

- Uber의 매칭 시스템은 "두 명의 기사가 같은 콜을 동시에 수락할 수 없다"는 강한 일관성(strict consistency)을 명시적 요구사항으로 둔다. ([Designing a Real-Time Ride-Hailing System Architecture - Uber](https://ashutoshkumars1ngh.medium.com/designing-a-real-time-ride-hailing-system-architecture-uber-643ca23c863f))
- 국내 배달 플랫폼(배민)의 사례에서도 라이더가 여러 건을 동시에 처리할 수 있는 구조라 동시 배정 경합이 상시적으로 발생하며, 이를 처리하는 로직의 복잡도가 빠르게 증가한다. ([배달아~ 배달 가는길 알려줘! - 우아한형제들 기술 블로그](https://woowabros.github.io/experience/2019/02/07/real-distance-finder.html))
- 이런 "선착순 자원 획득" 문제는 이커머스의 선착순 쿠폰 발급, 좌석 예매 등에서도 똑같이 나타나며, 실무에서는 **분산 락(Distributed Lock)** 으로 해결하는 것이 표준적인 패턴이다. ([Redisson 분산락을 이용한 동시성 제어](https://velog.io/@hgs-study/redisson-distributed-lock))

---

## 2. 시나리오

> 당신은 지역 대리운전 중개 플랫폼 "D-Call"의 백엔드 개발팀에 합류했다.
>
> D-Call은 손님이 호출하면 반경 내 대리기사 5명에게 동시에 알림을 보내고, 그중 가장 먼저 "수락" 버튼을 누른 기사에게 배정된다.
>
> 베타 테스트 중 사고가 발생했다.
> - 기사 두 명이 거의 동시에 수락 버튼을 눌렀는데, **둘 다 "배정되었습니다" 화면을 봤다.** 한 명은 현장에 도착했지만 이미 다른 기사가 손님을 태우고 떠난 뒤였다.
> - 반대로 어떤 콜은 5명 모두 30초 안에 응답하지 않았는데도 시스템이 "배정 대기 중" 상태로 멈춰 있어, 운영팀이 수동으로 재호출해야 했다.
>
> 팀 리드가 말했다.
> "이번 스프린트 목표는 두 가지예요. 첫째, 동시에 여러 명이 수락해도 시스템은 정확히 한 명만 배정할 것. 둘째, 아무도 응답하지 않으면 자동으로 다음 단계(재호출/확대)로 넘어갈 것."

---

## 3. 핵심 기능 명세

| 기능 | 설명 |
|---|---|
| 콜 생성 API | `POST /api/calls` — 승객 위치 기반으로 콜 생성, 주변 기사 후보 목록 조회 |
| 기사 응답 API | `POST /api/calls/{callId}/accept` — 기사가 수락 시 호출. **동시에 여러 기사가 호출해도 단 하나의 요청만 성공** 해야 함 |
| 단일 배정 보장 | **Redisson 분산 락**으로 `callId` 단위 락을 잡고, 락 획득에 성공한 요청만 배정 처리(DB 상태를 `ASSIGNED`로 전이) 후 락 해제. 락 획득 실패/이미 배정된 콜에 대한 응답은 `409 Conflict` 반환 |
| 배정 상태 전이 | 콜 상태 머신: `WAITING`(대기) → `ASSIGNED`(배정) → `IN_PROGRESS`(진행) → `COMPLETED`(완료), 예외 경로로 `TIMEOUT`(무응답) → `EXPANDED`(재호출) |
| 응답 타임아웃 처리 | 콜 생성 후 N초(예: 30초) 내 아무도 응답하지 않으면 상태를 `TIMEOUT`으로 전이하고 반경을 넓혀 재호출(`EXPANDED`) — 스케줄러 또는 지연 큐로 구현 |
| 배정 이력 조회 | `GET /api/calls/{callId}` — 콜의 상태 전이 이력(누가 언제 수락했는지, 실패한 시도 포함) 확인 |
| 동시성 테스트 | 동일 콜에 대해 다수의 기사 응답을 동시에 발사하는 테스트(k6/JMeter 또는 멀티스레드 테스트 코드)로 "정확히 1건만 성공"을 검증 |

---

## 4. 아키텍처

### 4.1 시스템 구성도 (텍스트)

```
[기사 앱 다수] ──(POST /calls/{id}/accept, 거의 동시)──▶ [Spring Boot 4 API 서버]
                                                              │
                                                    ┌─────────┴─────────┐
                                                    ▼                   ▼
                                    [Amazon ElastiCache for Redis]   [Amazon RDS]
                                    - Redisson 분산 락(callId 단위)   - Call, CallAssignmentAttempt
                                                    │
                                                    ▼
                                    [락 획득 성공 시에만 DB 상태 전이 커밋]
                                                    │
                                                    ▼
                                    [배정 결과를 승객/기사 앱에 응답 또는 이벤트 전파]
```

### 4.2 데이터 모델 개요

- **Call**(콜) — id, riderId, originLat/Lng, status(WAITING/ASSIGNED/IN_PROGRESS/COMPLETED/TIMEOUT/EXPANDED), createdAt, expiresAt
- **CallAssignmentAttempt**(배정 시도 이력) — id, callId, driverId, result(SUCCESS/FAILED_ALREADY_ASSIGNED/FAILED_EXPIRED), attemptedAt — "왜 이 기사는 실패했는지" 추적용
- **Driver**(기사) — id, name, status(ONLINE/OFFLINE/ON_TRIP)

### 4.3 기술 선택과 근거

| 요소 | 선택 | 근거 |
|---|---|---|
| 이중 배정 방지 | **Redisson 분산 락** (`callId` 기준) | Redisson은 Pub/Sub 구조로 락 해제를 대기 클라이언트에 즉시 알려, 스핀 락 대비 효율적으로 대기·재시도 가능 |
| 락 대안 검토 | DB 비관적 락(`SELECT ... FOR UPDATE`)도 가능하나, 다수 API 서버 인스턴스로 수평 확장할 계획이 있다면 애플리케이션 레벨의 분산 락이 더 일관된 방식 | |
| 상태 전이 관리 | 명시적 상태 머신(Enum + 상태 전이 검증 로직) | 잘못된 상태 전이(예: 이미 `COMPLETED`인 콜을 다시 `ASSIGNED`로)를 코드 레벨에서 차단 |
| 타임아웃 처리 | 스케줄러(`@Scheduled`) 또는 지연 큐 | 별도 인프라 없이 구현 가능한 가장 단순한 방식으로 시작, 확장 시 SQS 지연 메시지로 전환 가능 |
| 동시 처리 성능 | Spring Boot 4 가상 스레드 | 배차 API는 Redis/DB I/O 대기가 대부분이라 동시 요청 처리량 확보에 유리 |

- 참고: [Redisson 분산락을 이용한 동시성 제어](https://velog.io/@hgs-study/redisson-distributed-lock)
- 참고: [Redis 분산 락을 활용한 동시성 처리 - miintto.log](https://miintto.github.io/docs/distributed-lock)
- 참고: [재고 관리부터 티켓 예매까지, 동시성 이슈 박살내기: Redis 분산 락 실전 가이드](https://blog.leaphop.co.kr/blogs/98/%EC%9E%AC%EA%B3%A0_%EA%B4%80%EB%A6%AC%EB%B6%80%ED%84%B0_%ED%8B%B0%EC%BC%93_%EC%98%88%EB%A7%A4%EA%B9%8C%EC%A7%80__%EB%8F%99%EC%8B%9C%EC%84%B1_%EC%9D%B4%EC%8A%88_%EB%B0%95%EC%82%B4%EB%82%B4%EA%B8%B0__Redis_%EB%B6%84%EC%82%B0_%EB%9D%BD_%EC%8B%A4%EC%A0%84_%EA%B0%80%EC%9D%B4%EB%93%9C)

### 4.4 동시성 검증 방법 (팀 발표에서 강력한 증명 포인트)

- 같은 `callId`에 대해 5개의 기사 응답 요청을 **정확히 같은 시각**에 발사하는 테스트 코드 작성(`ExecutorService` + `CountDownLatch` 활용)
- 분산 락 적용 전/후를 비교해 "적용 전에는 2건 이상 성공, 적용 후에는 항상 1건만 성공"함을 로그·테스트 결과로 시연

---

## 5. 개발 로드맵 (실제 개발 가능 기간 3~4주 기준)

> 프로그램 일정 기준: 6주차(07.27~07.31) 기획 마무리 → **7주차(08.03~08.07) / 8주차(08.10~08.14) / 9주차(08.17~08.21)** 3주간 개발 → 9주차 팀밋업(08.18) 중간 점검 → 10주차 성과교류회(08.25) 최종 발표.

### 7주차 (08.03~08.07) — 도메인 모델 및 기본 API

- Day 1~2: `Call`, `Driver`, `CallAssignmentAttempt` 엔티티 및 상태 머신 설계
- Day 3~4: 콜 생성 API, 기사 응답 API(락 미적용 상태) 구현
- Day 5: 분산 락 미적용 상태에서 동시 요청 시 이중 배정이 실제로 재현됨을 테스트로 확인(문제를 먼저 증명)
- **완료 기준**: 동시 요청 테스트에서 이중 배정이 재현되는 것을 로그로 확인

### 8주차 (08.10~08.14) — 분산 락 적용 및 상태 전이

- Day 1~3: Redisson 연동, `callId` 단위 분산 락 적용, 락 획득 실패 시 처리(409 응답) 구현
- Day 4~5: 동시성 테스트 재실행 — "정확히 1건만 성공"을 검증, 상태 전이 로직(WAITING→ASSIGNED→IN_PROGRESS→COMPLETED) 완성
- **완료 기준**: 동일한 동시 요청 테스트에서 정확히 1건만 성공함을 재현·검증

### 9주차 (08.17~08.21) — 타임아웃 처리, 통합, 발표 준비

- Day 1~2: 응답 타임아웃(무응답 시 `TIMEOUT`→`EXPANDED`) 스케줄러 구현
- Day 3: 배정 이력 조회 API, 관리자용 콜 현황 대시보드(간단한 조회 화면)
- Day 4: AWS 배포(EC2/Elastic Beanstalk + ElastiCache), 부하/동시성 테스트 결과 정리
- Day 5(팀밋업 08.18 전후): "적용 전/후 비교"를 중심으로 한 데모 시나리오 및 발표자료 준비
- **완료 기준**: 동시성 문제 재현 → 해결 과정을 라이브 데모로 보여줄 수 있는 상태

---

## 6. 참고 자료

- [Designing a Real-Time Ride-Hailing System Architecture - Uber](https://ashutoshkumars1ngh.medium.com/designing-a-real-time-ride-hailing-system-architecture-uber-643ca23c863f)
- [배달아~ 배달 가는길 알려줘! - 우아한형제들 기술 블로그](https://woowabros.github.io/experience/2019/02/07/real-distance-finder.html)
- [배달의민족의 인공지능 배차는 어떻게 작동하나 - 바이라인네트워크](https://byline.network/2020/12/20-111/)
- [Redisson 분산락을 이용한 동시성 제어](https://velog.io/@hgs-study/redisson-distributed-lock)
- [Redis 분산 락을 활용한 동시성 처리 - miintto.log](https://miintto.github.io/docs/distributed-lock)
- [재고 관리부터 티켓 예매까지, 동시성 이슈 박살내기: Redis 분산 락 실전 가이드](https://blog.leaphop.co.kr/blogs/98/%EC%9E%AC%EA%B3%A0_%EA%B4%80%EB%A6%AC%EB%B6%80%ED%84%B0_%ED%8B%B0%EC%BC%93_%EC%98%88%EB%A7%A4%EA%B9%8C%EC%A7%80__%EB%8F%99%EC%8B%9C%EC%84%B1_%EC%9D%B4%EC%8A%88_%EB%B0%95%EC%82%B4%EB%82%B4%EA%B8%B0__Redis_%EB%B6%84%EC%82%B0_%EB%9D%BD_%EC%8B%A4%EC%A0%84_%EA%B0%80%EC%9D%B4%EB%93%9C)
- [에스코어 | Redis를 활용한 안전하게 동시성 이슈 제어하기](https://s-core.co.kr/insight/view/redis%eb%a5%bc-%ed%99%9c%ec%9a%a9%ed%95%9c-%ec%95%88%ec%a0%84%ed%95%98%ea%b2%8c-%eb%8f%99%ec%8b%9c%ec%84%b1-%ec%9d%b4%ec%8a%88-%ec%a0%9c%ec%96%b4%ed%95%98%ea%b8%b0/)
