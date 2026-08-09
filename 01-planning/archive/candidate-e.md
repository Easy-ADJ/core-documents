# EV 충전기 관제 시스템 (경량 OCPP 스타일) — 상세 기획 (후보 E)

> 📦 **보관 문서 — 현행 아님.** 미채택 후보의 기획 초안입니다. 현재 기획은 [service-spec.md](../service-spec.md), 현행 아키텍처는 [architecture.md](../../02-design/architecture.md)를 참조하세요.

> [project-candidates.md](./project-candidates.md)의 **후보 E(EV 충전기 관제 시스템)** — 문제 5 "상태 기반 디바이스와의 실시간 프로토콜 동기화" 대응 — 를 실제 개발에 들어갈 수 있는 수준으로 구체화한 문서다. 기술 스택은 Java 17+ / **Spring Boot 4** / **AWS** 기준.
>
> ⚠️ 프로토콜을 직접 설계해야 하고 연결 끊김·재동기화 같은 예외 시나리오가 많아 6개 후보 중 난이도가 높은 편이다.

---

## 1. 문제 재정의

전기차 충전기처럼 "지금 이 장치가 충전 중인지/대기 중인지/고장인지"를 서버와 실시간으로 동기화해야 하는 상태 기반(stateful) 디바이스 제어는 실무에서 다음과 같은 어려움을 동반한다.

- 업계 표준 프로토콜인 **OCPP(Open Charge Point Protocol)** 는 충전소-중앙관리시스템(CSMS) 간 WebSocket 연결에서 BootNotification(초기 등록), Heartbeat(생존 확인), 상태 보고, 원격 명령(시작/정지/Reset) 등을 정의한다. ([OCPP 1.6J — Open Charge Point Protocol](https://ocpp.md/ocpp-1.6j/))
- 연결이 끊기면 충전기는 지수 백오프(exponential backoff)로 재연결을 시도해야 하고, 재연결 후에는 트랜잭션 관련 메시지(StartTransaction, StopTransaction, MeterValues)를 큐에 쌓아뒀다가 순서대로 재전송해야 한다. ([OCPP 2.0.1 Part 4 - JSON over WebSockets implementation guide](https://downloads.regulations.gov/FHWA-2022-0008-0404/attachment_1.pdf))
- 네트워크가 불안정한 현장에서는 "서버는 충전 중으로 알고 있는데 실제로는 끊겨서 멈춰 있는" 상태 불일치가 흔히 발생한다.

---

## 2. 시나리오

> 당신은 EV 충전 인프라 운영사 "D-Charge"의 백엔드 개발팀에 합류했다.
>
> D-Charge는 각 충전기가 서버와 WebSocket으로 연결되어 상태(대기/충전중/고장)를 실시간으로 보고하고, 관제팀이 대시보드에서 원격으로 충전 시작/정지 명령을 내릴 수 있다.
>
> 현장 테스트 중 문제가 발생했다.
> - 지하 주차장에 설치된 충전기의 네트워크가 잠깐 끊겼다가 다시 붙었는데, 서버 대시보드에는 여전히 "충전 중"으로 표시되어 있었다. 실제로는 충전이 끝나 있었지만, 다음 사용자가 "사용 중"이라는 안내를 보고 다른 충전기로 발길을 돌렸다.
> - 관리자가 원격으로 "충전 정지" 명령을 보냈는데, 마침 그 순간 충전기 연결이 끊겨 있어 명령이 유실됐다. 관리자는 명령이 실행된 줄 알았지만 실제로는 반영되지 않았다.
>
> 팀 리드가 말했다.
> "이번 스프린트 목표는 두 가지예요. 첫째, 연결이 끊겼다 다시 붙으면 서버와 충전기의 상태가 반드시 다시 맞춰질 것. 둘째, 연결이 끊긴 상태로 보낸 명령은 '실패'로 명확히 알 수 있을 것."

---

## 3. 핵심 기능 명세

| 기능 | 설명 |
|---|---|
| 연결 등록(BootNotification 유사) | 충전기(시뮬레이터)가 WebSocket 연결 직후 자신의 식별자·모델 정보를 서버에 전송, 서버는 이를 승인하고 Heartbeat 주기를 응답 |
| Heartbeat | 충전기가 일정 주기로 생존 신호(ping)를 전송, 서버는 일정 시간 미수신 시 해당 충전기를 `OFFLINE`으로 표시 |
| 상태 보고 | 충전기가 상태 변화(`AVAILABLE`/`CHARGING`/`FAULTED`)를 서버에 push, 서버는 이를 즉시 저장하고 대시보드에 반영 |
| 원격 명령 | 관리자가 `POST /api/chargers/{chargerId}/commands`로 시작/정지 명령을 보내면, 서버가 해당 충전기의 WebSocket 세션으로 명령을 전달. **연결이 없는 상태면 즉시 `실패` 응답**(명령이 유실된 채 "성공"으로 보이지 않도록) |
| 재연결 시 상태 재동기화 | 재연결 직후, 서버가 마지막으로 알고 있는 상태와 충전기가 보고하는 실제 상태를 비교해 불일치 시 서버 상태를 충전기 보고값으로 갱신 |
| 트랜잭션 메시지 큐잉 | 연결이 끊긴 동안 발생한 충전 시작/종료 이벤트는 충전기 측에서 큐에 쌓아두고, 재연결 시 순서대로 재전송(시뮬레이터에서 재현) |
| 관제 대시보드 조회 API | `GET /api/chargers` — 전체 충전기의 현재 상태(온라인/오프라인, 충전 상태) 목록 조회 |

---

## 4. 아키텍처

### 4.1 시스템 구성도 (텍스트)

```
[충전기 시뮬레이터 다수] ◀──WebSocket 연결(양방향)──▶ [Spring Boot 4 WebSocket 서버]
      │  - BootNotification                                  │
      │  - Heartbeat(주기적)                                  ├─ 세션 관리자(chargerId ↔ WebSocket Session)
      │  - StatusNotification                                 ├─ 상태 머신(AVAILABLE/CHARGING/FAULTED/OFFLINE)
      │  - (연결 끊김 시 지수 백오프 재연결)                     └─ 원격 명령 라우팅
      ▼                                                        │
[재연결 후 큐잉된 메시지 재전송]                                  ▼
                                                        [Amazon RDS]
                                                        - Charger, ChargerStatusHistory, RemoteCommand
```

### 4.2 데이터 모델 개요

- **Charger**(충전기) — id, chargerCode, currentStatus(AVAILABLE/CHARGING/FAULTED/OFFLINE), lastHeartbeatAt
- **ChargerStatusHistory**(상태 변경 이력) — id, chargerId, status, changedAt, source(DEVICE_REPORTED/RESYNC_ON_RECONNECT)
- **RemoteCommand**(원격 명령 이력) — id, chargerId, commandType(START/STOP), result(SENT/FAILED_NO_CONNECTION/ACKED), requestedAt

### 4.3 기술 선택과 근거

| 요소 | 선택 | 근거 |
|---|---|---|
| 통신 방식 | Spring WebSocket (양방향 지속 연결) | 서버가 충전기에 즉시 명령을 보내야 하므로 REST 폴링이 아닌 지속 연결이 필수 |
| 프로토콜 설계 | OCPP 핵심 개념(BootNotification/Heartbeat/StatusNotification)을 단순화해 자체 메시지 포맷으로 구현 | 실제 OCPP 풀 스펙은 매우 방대해 3~4주 내 완주가 어려움. 핵심 개념만 재현해 실무 문제의 본질(상태 동기화, 재연결)에 집중 |
| Heartbeat 판정 | 마지막 Heartbeat 수신 시각 + 임계 시간 초과 시 `OFFLINE` 전이 | OCPP 표준도 Heartbeat 간격을 서버가 지정하고 미수신 시 연결 이상으로 판단하는 방식을 사용 |
| 재연결 시 재동기화 | 재연결 직후 서버가 최신 상태를 다시 요청(또는 충전기가 자동으로 현재 상태를 재전송) | "서버가 알고 있는 상태"와 "실제 장치 상태"가 다를 수 있다는 전제 하에, 재연결을 상태 정합성을 다시 맞추는 시점으로 명확히 정의 |
| 명령 실패 처리 | 연결 없는 충전기에 대한 명령은 즉시 실패 응답 | "명령을 보냈다"와 "명령이 실행됐다"를 구분해, 관리자가 잘못된 성공 신호를 받지 않도록 함 |

- 참고: [OCPP 1.6J — Open Charge Point Protocol 1.6 (JSON over WebSocket)](https://ocpp.md/ocpp-1.6j/)
- 참고: [OCPP 2.0.1 Part 4 - JSON over WebSockets implementation guide](https://downloads.regulations.gov/FHWA-2022-0008-0404/attachment_1.pdf)

### 4.4 재연결·상태 정합성 전략

- 충전기 WebSocket 클라이언트(시뮬레이터)는 연결 끊김 감지 시 랜덤 지연을 섞은 지수 백오프로 재연결을 시도한다.
- 연결 끊긴 동안 발생한 트랜잭션 이벤트(충전 시작/종료)는 로컬 큐에 적재했다가, 재연결 성공 후 발생 순서대로 서버에 재전송한다.
- 서버는 재동기화 이벤트를 `ChargerStatusHistory`에 `source=RESYNC_ON_RECONNECT`로 별도 기록해, "언제 상태가 자동으로 바로잡혔는지" 추적 가능하게 한다.

---

## 5. 개발 로드맵 (실제 개발 가능 기간 3~4주 기준)

> 프로그램 일정 기준: 6주차(07.27~07.31) 기획 마무리 → **7주차(08.03~08.07) / 8주차(08.10~08.14) / 9주차(08.17~08.21)** 3주간 개발 → 9주차 팀밋업(08.18) 중간 점검 → 10주차 성과교류회(08.25) 최종 발표.

### 7주차 (08.03~08.07) — WebSocket 연결 관리 및 기본 프로토콜

- Day 1~2: `Charger`, `ChargerStatusHistory` 도메인 설계, WebSocket 서버 기본 구조 구현
- Day 3~4: BootNotification 유사 등록 메시지, Heartbeat 수신·판정 로직 구현
- Day 5: 상태 보고(StatusNotification) 메시지 처리 및 저장
- **완료 기준**: 시뮬레이터를 연결하면 서버가 등록을 승인하고, Heartbeat가 끊기면 일정 시간 후 `OFFLINE`으로 전이됨

### 8주차 (08.10~08.14) — 원격 명령 및 연결 끊김 시나리오

- Day 1~2: 원격 명령 API 및 WebSocket 세션을 통한 명령 전달 구현
- Day 3: 연결 없는 충전기에 명령 시도 시 즉시 실패 처리 구현
- Day 4~5: 시뮬레이터에서 의도적으로 연결을 끊었다 재연결하는 시나리오 구현, 재연결 시 상태 재동기화 로직 완성
- **완료 기준**: 연결 끊김 → 재연결 시나리오에서 서버 상태가 충전기의 실제 상태와 다시 일치함을 확인

### 9주차 (08.17~08.21) — 트랜잭션 큐잉, 대시보드, 발표 준비

- Day 1~2: 연결 끊긴 동안의 트랜잭션 이벤트 큐잉·재전송(시뮬레이터 측) 구현
- Day 3: 관제 대시보드용 조회 API(`GET /api/chargers`) 및 간단한 화면 구성
- Day 4: AWS 배포(EC2/Elastic Beanstalk + RDS), 다수 충전기 시뮬레이션으로 통합 테스트
- Day 5(팀밋업 08.18 전후): "연결 끊김 → 잘못된 상태 표시 → 재연결 후 자동 정정" 시연 시나리오 및 발표자료 준비
- **완료 기준**: 네트워크 단절 상황을 라이브로 재현하고, 재동기화 과정을 시연 가능

---

## 6. 참고 자료

- [OCPP 소개 - 한국스마트그리드협회](https://www.ksga.org/cert/ocpp/01.do)
- [OCPP 1.6J — Open Charge Point Protocol 1.6 (JSON over WebSocket)](https://ocpp.md/ocpp-1.6j/)
- [Complete OCPP 1.6 WebSocket Communication Developer Guide](https://gist.github.com/ChxGuillaume/a3d072cf711a196459e7ac9e5d5bb446)
- [OCPP 2.0.1 Part 4 - JSON over WebSockets implementation guide](https://downloads.regulations.gov/FHWA-2022-0008-0404/attachment_1.pdf)
- [What Is OCPP? Open Charge Point Protocol Explained - OCPPLab](https://www.ocpplab.com/blog/what-is-ocpp)
- [KR102312946B1 - OCPP 프로토콜을 적용한 전기 자동차 충전 인프라 운영 관리 시스템](https://patents.google.com/patent/KR102312946B1/ko)
