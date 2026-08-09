# IoT 텔레메틱스 수집 파이프라인 — 상세 기획 (후보 C)

> 📦 **보관 문서 — 현행 아님.** 미채택 후보의 기획 초안입니다. 현재 기획은 [service-spec.md](../service-spec.md), 현행 아키텍처는 [architecture.md](../../02-design/architecture.md)를 참조하세요.

> [project-candidates.md](./project-candidates.md)의 **후보 C(IoT 텔레메틱스 수집 파이프라인)** — 문제 2 "IoT 텔레메틱스 데이터 수집 파이프라인의 신뢰성" 대응 — 를 실제 개발에 들어갈 수 있는 수준으로 구체화한 문서다. 기술 스택은 Java 17+ / **Spring Boot 4** / **AWS** 기준.
>
> ⚠️ 팀 전체가 AWS IoT Core를 처음 접한다면 6개 후보 중 러닝커브가 가장 큰 편이다. AWS 지원 자원을 적극 활용하고 싶고 학습 속도에 자신 있는 팀에 추천한다.

---

## 1. 문제 재정의

차량·충전기 등 이기종 IoT 단말에서 올라오는 데이터를 손실 없이 수집·정제하는 것은 모빌리티 백엔드의 상시적인 과제다.

- GPS 신호 반사로 인한 위치 튐(도로 밖으로 순간 이동한 것처럼 보이는 현상), 터널·도심 밀집지역에서의 신호 완전 소실 등 **"데이터 정제·정규화" 자체가 상시 과제**다. ([The unified MQTT Platform for Fleet Telematics & IoT - EMQX](https://www.emqx.com/en/solutions/fleet-telematics))
- MQTT는 대량 기기 연결에는 적합하지만, 대용량 스트림을 실시간으로 처리하기에는 한계가 있어 Kafka/Kinesis 같은 스트리밍 플랫폼과 결합하는 경우가 많다.
- 쏘카는 AWS IoT Core로 수집한 차량 데이터를 Amazon MSK(Kafka)로 흘려보내, 여러 비즈니스 도메인이 각자 컨슈머를 붙여 쓰는 구조를 택했다. ([차량용 단말을 위한 IoT 파이프라인 구축기 #1 - SOCAR Tech Blog](https://tech.socarcorp.kr/mobility/2022/01/06/socar-iot-pipeline-1.html))

---

## 2. 시나리오

> 당신은 화물차 관제 서비스 "D-Telemetry"의 백엔드 개발팀에 합류했다.
>
> 차량마다 부착된 단말이 위치·속도·연료 잔량 데이터를 주기적으로 전송하면, 서버는 이를 수집해 저장하고 관제팀이 조회할 수 있게 한다.
>
> 파일럿 운영 중 문제가 발견됐다.
> - 특정 구간(지하 주차장, 터널)을 지날 때 단말이 잘못된 좌표(도로 밖, 심지어 바다 위)를 보내는 경우가 있었는데, 이 값이 그대로 저장되어 관제팀이 "차량이 실종됐다"고 오인하는 소동이 있었다.
> - 단말 100대가 한꺼번에 데이터를 보내는 상황에서 수집 서버가 느려지며 일부 데이터가 유실됐다.
>
> 팀 리드가 말했다.
> "이번 스프린트에서는 두 가지를 만들어야 해요. 첫째, 명백히 잘못된 좌표는 '원본은 보존하되' 정제된 데이터에서는 걸러낼 것. 둘째, 단말이 몰려서 데이터를 보내도 유실 없이 다 받아낼 것."

---

## 3. 핵심 기능 명세

| 기능 | 설명 |
|---|---|
| 단말 데이터 수신 | 차량 단말(시뮬레이터)이 **MQTT**로 좌표·속도·연료 데이터를 발행 → **AWS IoT Core**가 브로커 역할 |
| 라우팅 | AWS IoT Core **Rule Engine**이 수신 메시지를 Kinesis Data Streams로 라우팅(원본은 동시에 S3에도 적재해 원본 보존) |
| 스트림 소비 | Spring Boot 4 애플리케이션이 Kinesis Data Streams를 구독(Consumer)해 실시간으로 메시지 처리 |
| 이상치 필터링 | 정제 규칙 적용 — ①이동 불가능한 속도로 튄 좌표(직전 좌표 대비 물리적으로 불가능한 거리) 제외, ②도로 반경을 크게 벗어난 좌표 플래그 처리, ③결측/중복 타임스탬프 제거 |
| 원본/정제 데이터 분리 저장 | 원본(raw)은 S3에 그대로 보존(감사·재처리용), 정제된(clean) 데이터만 RDS에 저장해 조회 API에 노출 |
| 정제 결과 조회 API | `GET /api/vehicles/{vehicleId}/telemetry?from=&to=` — 정제된 데이터만 반환 |
| 이상치 리포트 API | `GET /api/telemetry/anomalies` — 필터링된(제외된) 데이터 건수와 사유를 관제팀이 확인할 수 있는 API — "걸러낸 게 맞는지" 검증 가능하게 함 |

---

## 4. 아키텍처

### 4.1 시스템 구성도 (텍스트)

```
[차량 단말 시뮬레이터 다수] ──MQTT Publish──▶ [AWS IoT Core]
                                                   │
                                       [IoT Core Rule Engine]
                                        (조건에 따라 다중 경로로 라우팅)
                                     ┌────────────┴────────────┐
                                     ▼                          ▼
                     [Amazon Kinesis Data Streams]      [Amazon S3 (원본 raw 적재)]
                     (실시간 처리 경로)                   (Kinesis Firehose 경유, 감사/재처리용)
                                     │
                                     ▼
                     [Spring Boot 4 Consumer]
                     - 이상치 필터링 로직
                     - 정제 데이터 저장
                                     │
                                     ▼
                     [Amazon RDS] ── 정제 데이터, 조회 API 제공
```

- AWS 공식 레퍼런스는 IoT Core Rule을 통해 데이터를 Kinesis Data Streams(실시간)와 Firehose→S3(배치/원본 보존)로 동시에 라우팅하는 "이중 경로(dual-path)" 구성을 권장한다. ([Architect a dual-path IoT conversation analytics solution on AWS](https://geekfence.com/architect-a-dual-path-iot-conversation-analytics-solution-on-aws/))

### 4.2 데이터 모델 개요

- **RawTelemetryEvent**(S3, 파일 형태) — deviceId, lat, lng, speed, fuel, receivedAt (원본 그대로)
- **CleanTelemetry**(RDS) — id, vehicleId, lat, lng, speed, fuel, recordedAt
- **TelemetryAnomaly**(RDS) — id, vehicleId, rawPayload, reason(SPEED_IMPOSSIBLE/OUT_OF_BOUNDS/DUPLICATE_TIMESTAMP), detectedAt

### 4.3 기술 선택과 근거

| 요소 | 선택 | 근거 |
|---|---|---|
| 단말-서버 프로토콜 | MQTT + AWS IoT Core | 대량의 경량 기기 연결에 최적화된 관리형 브로커, 인증서 기반 보안 연결 기본 제공 |
| 스트림 처리 | Amazon Kinesis Data Streams | IoT Core Rule Action으로 바로 연동 가능하며, 수집과 처리를 분리(디커플링)해 처리 지연이 수집 유실로 이어지지 않도록 함 |
| 원본 보존 | Amazon S3 (Firehose 경유) | 정제 로직에 버그가 있어도 원본이 남아있어 재처리 가능 |
| 이상치 필터링 로직 | 애플리케이션 레벨 규칙 기반 필터 | 별도 ML 없이 "직전 좌표 대비 물리적으로 불가능한 이동"만 걸러내는 단순 규칙으로도 데모 목적에는 충분 |
| 정제 데이터 저장 | Amazon RDS | 조회 API가 표준 SQL로 단순하게 동작 |

- 참고: [Ingesting enriched IoT data into Amazon S3 using Amazon Kinesis Data Firehose](https://aws.amazon.com/blogs/iot/ingesting-enriched-iot-data-into-amazon-s3-using-amazon-kinesis-data-firehose/)
- 참고: [Sensor Data Processing on AWS using IoT Core, Kinesis and ElastiCache](https://dev.to/frosnerd/sensor-data-processing-on-aws-using-iot-core-kinesis-and-elasticache-26j1)

### 4.4 신뢰성 전략

- IoT Core의 Basic Ingest를 활용하면 메시지 브로커 경로를 거치지 않고 바로 Kinesis/S3로 라우팅해 대량 수집 시 비용·지연을 줄일 수 있다.
- Kinesis Consumer는 Checkpoint를 활용해 처리 도중 장애가 발생해도 마지막 처리 지점부터 재개 가능하도록 구현한다.

---

## 5. 개발 로드맵 (실제 개발 가능 기간 3~4주 기준)

> 프로그램 일정 기준: 6주차(07.27~07.31) 기획 마무리 → **7주차(08.03~08.07) / 8주차(08.10~08.14) / 9주차(08.17~08.21)** 3주간 개발 → 9주차 팀밋업(08.18) 중간 점검 → 10주차 성과교류회(08.25) 최종 발표.

### 7주차 (08.03~08.07) — AWS IoT Core 연동 확인

- Day 1~2: AWS IoT Core 학습, Thing/인증서 등록, MQTT 발행 테스트 클라이언트 작성
- Day 3~4: IoT Core Rule Engine 설정 — Kinesis Data Streams로 라우팅 규칙 구성
- Day 5: 간단한 페이로드로 IoT Core → Kinesis까지 데이터가 실제로 흐르는지 확인
- **완료 기준**: MQTT로 보낸 메시지가 Kinesis 콘솔에서 확인됨

### 8주차 (08.10~08.14) — Spring Boot 컨슈머 및 정제 로직

- Day 1~2: Spring Boot 4 Kinesis 컨슈머 구현(AWS SDK), 수신 메시지 파싱
- Day 3~4: 이상치 필터링 규칙 구현(속도 기반, 경계 기반), `CleanTelemetry`/`TelemetryAnomaly` 저장 분리
- Day 5: 원본 S3 적재 경로(Firehose) 연동 확인
- **완료 기준**: 의도적으로 이상 좌표를 섞어 보냈을 때, 정제 데이터에서는 제외되고 이상치 리포트에는 기록됨을 확인

### 9주차 (08.17~08.21) — 조회 API, 통합, 발표 준비

- Day 1~2: 정제 데이터 조회 API, 이상치 리포트 API 구현
- Day 3: 다수 단말 시뮬레이터로 부하 테스트, 처리 지연/유실 여부 측정
- Day 4: 전체 파이프라인 AWS 배포 점검, CloudWatch로 처리 지표 모니터링 대시보드 구성
- Day 5(팀밋업 08.18 전후): "이상 데이터 주입 → 자동 필터링" 시연 시나리오 및 발표자료 준비
- **완료 기준**: MQTT 발행부터 정제 데이터 조회까지 전체 파이프라인을 라이브로 시연 가능

---

## 6. 참고 자료

- [The unified MQTT Platform for Fleet Telematics & IoT - EMQX](https://www.emqx.com/en/solutions/fleet-telematics)
- [차량용 단말을 위한 IoT 파이프라인 구축기 #1 - SOCAR Tech Blog](https://tech.socarcorp.kr/mobility/2022/01/06/socar-iot-pipeline-1.html)
- [Tracking Assets & Locating Devices Using AWS IoT (공식 레퍼런스 아키텍처)](https://docs.aws.amazon.com/solutions/tracking-assets-and-locating-devices-using-aws-iot/)
- [Architect a dual-path IoT conversation analytics solution on AWS](https://geekfence.com/architect-a-dual-path-iot-conversation-analytics-solution-on-aws/)
- [Ingesting enriched IoT data into Amazon S3 using Amazon Kinesis Data Firehose](https://aws.amazon.com/blogs/iot/ingesting-enriched-iot-data-into-amazon-s3-using-amazon-kinesis-data-firehose/)
- [Sensor Data Processing on AWS using IoT Core, Kinesis and ElastiCache](https://dev.to/frosnerd/sensor-data-processing-on-aws-using-iot-core-kinesis-and-elasticache-26j1)
- [Kinesis Data Streams - AWS IoT Core 공식 문서](https://docs.aws.amazon.com/iot/latest/developerguide/kinesis-rule-action.html)
- [AWS IoT Core Improves the Ability to Ingest Large Amounts of Device Data at a Lower Cost](https://aws.amazon.com/about-aws/whats-new/2018/11/aws-iot-core-improves-ability-to-ingest-large-amounts-of-data)
