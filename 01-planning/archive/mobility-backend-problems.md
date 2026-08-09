# 모빌리티 백엔드 실무 문제 리스트업

> 📦 **보관 문서 — 현행 아님.** 주제 선정 과정의 기록입니다. 현재 기획은 [service-spec.md](../service-spec.md)를 참조하세요.

> ※ 아래 리서치는 특정 파트너 기업에 한정하지 않고 카카오모빌리티·티맵모빌리티·쏘카·우아한형제들(배민)·Uber 등 국내외 모빌리티/배달 플랫폼의 공개 기술 자료와 일반적인 백엔드 엔지니어링 자료를 기반으로 정리했다.

> 과정·기술 스택 개요는 [프로젝트 개요](../overview.md), 이 문제들을 프로젝트로 옮긴 결과는 [프로젝트 후보 선정](./project-candidates.md) 참조.

---

## 문제 1. 대규모 동시 위치 업데이트 처리 (실시간 위치 추적)

- **문제 정의**: 수천~수백만 대의 차량/기기가 수 초 단위로 GPS 좌표를 전송하고, 서버는 이를 실시간으로 저장·조회·전파해야 한다.
- **실무에서 어려운 이유**
  - Uber는 기사 앱과의 지속적인 연결(WebSocket)로 위치를 수신해 Redis 기반 지오스페이셜 캐시에 저장하고, 위치 갱신을 200ms 이내에 승객에게 전파해야 한다는 엄격한 지연시간 요구사항을 갖고 있다. 
  - 배터리 소모를 줄이기 위해 차량이 정지해 있을 때는 갱신 주기를 늦추는 적응형 샘플링도 필요하다.
- **근거 자료**:
  - [How Uber Scales Real-Time Location Tracking for Millions of Users](https://medium.com/@wahab9766/how-uber-scales-real-time-location-tracking-for-millions-of-users-ff5c43ab6632)
  - [How Uber's Real-Time Location Tracking System Works at Scale](https://www.bnxt.ai/blog/how-ubers-real-time-location-tracking-system-works-at-scale)
  - [카카오 T 택시의 배차 시스템 - 카카오모빌리티](https://www.kakaomobility.com/contents/taxi-dispatch) (매초 쏟아지는 호출 요청과 실시간 위치 처리 언급)

## 문제 2. IoT 텔레메틱스 데이터 수집 파이프라인의 신뢰성

- **문제 정의**: 차량/충전기 등 이기종 IoT 단말에서 올라오는 데이터(MQTT/TCP 등 프로토콜 혼재)를 손실 없이 수집·정제해야 한다.
- **실무에서 어려운 이유**
  - GPS 신호 반사로 인한 위치 튐(도로 밖으로 순간 이동하는 것처럼 보이는 현상), 터널·도심 밀집지역에서의 신호 완전 소실 등 "데이터 정제·정규화" 자체가 상시적인 과제다. 
  - MQTT는 대량 기기 연결에는 적합하지만 대용량 스트림 실시간 처리에는 한계가 있어 Kafka/Kinesis와 결합하는 경우가 많다. 
  - 쏘카는 AWS IoT Core로 수집한 차량 데이터를 Amazon MSK(Kafka)로 흘려보내 여러 비즈니스 도메인이 구독해 쓰는 구조를 택했다.
- **근거 자료**:
  - [The unified MQTT Platform for Fleet Telematics & IoT - EMQX](https://www.emqx.com/en/solutions/fleet-telematics) (GPS drift, 신호 손실, 프로토콜 이기종성)
  - [차량용 단말을 위한 IoT 파이프라인 구축기 #1 - SOCAR Tech Blog](https://tech.socarcorp.kr/mobility/2022/01/06/socar-iot-pipeline-1.html)
  - [Tracking Assets & Locating Devices Using AWS IoT (공식 레퍼런스 아키텍처)](https://docs.aws.amazon.com/solutions/tracking-assets-and-locating-devices-using-aws-iot/)

## 문제 3. 배차/매칭 시 동시성 제어 (이중 배정 방지)

- **문제 정의**: 하나의 호출(콜)에 여러 기사가 동시에 응답할 때, 정확히 한 명에게만 배정되어야 한다. 분산 환경에서 이 "정합성"을 어떻게 보장할지가 핵심 과제다.
- **실무에서 어려운 이유**
  - Uber의 매칭 시스템은 "두 명의 기사가 같은 콜을 동시에 수락할 수 없다"는 강한 일관성(strict consistency)을 요구사항으로 명시한다. 
  - 배민의 추천 배차는 비용 계산→경로 선택→라이더 선정→거리 계산의 다단계 파이프라인이며, 라이더가 여러 건을 동시 수행할 경우 조합의 수가 기하급수적으로 늘어나 빠른 연산이 어려워진다.
- **근거 자료**:
  - [Designing a Real-Time Ride-Hailing System Architecture - Uber](https://ashutoshkumars1ngh.medium.com/designing-a-real-time-ride-hailing-system-architecture-uber-643ca23c863f) (강한 일관성/분산 락 요구사항)
  - [배달아~ 배달 가는길 알려줘! - 우아한형제들 기술 블로그](https://woowabros.github.io/experience/2019/02/07/real-distance-finder.html)
  - [배달의민족의 인공지능 배차는 어떻게 작동하나 - 바이라인네트워크](https://byline.network/2020/12/20-111/)

## 문제 4. 결제/정산의 멱등성과 정합성

- **문제 정의**
  - 네트워크 타임아웃에 의한 재시도, 메시지 큐의 중복 전달, 사용자의 중복 클릭 등으로 동일 결제 요청이 여러 번 들어올 수 있으며, 이를 막지 못하면 이중 결제가 발생한다.
  - 또한 플랫폼-기사(또는 가맹점) 간 정산은 원장(ledger) 상에서 차변/대변 합이 항상 0이 되어야 하는 정합성이 요구된다.
- **실무에서 어려운 이유**
  - PG(결제대행사)와의 연동 시 발생하는 불일치를 없애기 위해 매일 야간 배치로 정산 내역을 대사(reconciliation)하는 과정이 필요하며, 이는 모빌리티 플랫폼의 기사 정산·차량 대여료 정산 등에도 동일하게 적용된다.
- **근거 자료**:
  - [Spring Boot로 구축하는 회복력 있는(Resilient) 결제 시스템](https://velog.io/@anlee/Spring-Boot%EB%A1%9C-%EA%B5%AC%EC%B6%95%ED%95%98%EB%8A%94-%ED%9A%8C%EB%B3%B5%EB%A0%A5-%EC%9E%88%EB%8A%94Resilient-%EA%B2%B0%EC%A0%9C-%EC%8B%9C%EC%8A%A4%ED%85%9C-%EC%B9%B4%EB%93%9C-%EA%B2%B0%EC%A0%9C%EB%B6%80%ED%84%B0-%EC%A0%95%EA%B8%B0-%EA%B2%B0%EC%A0%9C-%EC%9E%A5%EC%95%A0-%EB%8C%80%EC%9D%91%EA%B9%8C%EC%A7%80)
  - [쿠팡, 아마존, 알리와 같은 기업들은 결제 시스템을 어떻게 만드는걸까?](https://team-json-delivery.github.io/posts/pay-system/) (원장 구조·PSP 정산 대사)
  - [멱등성(Idempotency) 설계 가이드](https://velog.io/@2eunpal/%EB%A9%B1%EB%93%B1%EC%84%B1Idempotency-%EC%84%A4%EA%B3%84-%EA%B0%80%EC%9D%B4%EB%93%9C)

## 문제 5. 상태 기반 디바이스(충전기·차량)와의 실시간 프로토콜 동기화

- **문제 정의**: 전기차 충전기처럼 "지금 이 장치가 충전 중인지/대기 중인지/고장인지"를 서버와 실시간으로 동기화해야 하는 상태 기반(stateful) 디바이스 제어 문제.
- **실무에서 어려운 이유**: 업계 표준 프로토콜인 OCPP(Open Charge Point Protocol)는 충전소-중앙관리시스템 간 인증, 원격 충전 시작/정지, Reset, Firmware Update 등을 정의하며, 다수의 충전기를 그룹으로 묶어 실시간 상태를 하나의 서버에 지속적으로 반영해야 한다. 네트워크 단절, 재연결 시 상태 재동기화가 실무의 핵심 난제다.
- **근거 자료**:
  - [OCPP 소개 - 한국스마트그리드협회](https://www.ksga.org/cert/ocpp/01.do)
  - [KR102312946B1 - OCPP 프로토콜을 적용한 전기 자동차 충전 인프라 운영 관리 시스템](https://patents.google.com/patent/KR102312946B1/ko)

## 문제 6. 대용량 이벤트 기반 알림/상태 전파

- **문제 정의**: "배차 완료", "차량 도착", "충전 완료" 같은 이벤트를 다수의 클라이언트(승객 앱, 기사 앱, 관제 대시보드)에 실시간으로, 유실 없이 전파해야 한다.
- **실무에서 어려운 이유**
  - 폴링 방식은 서버 부하가 크고 지연이 발생하며, 단순 WebSocket 브로드캐스트만으로는 서버 재시작·장애 시 이벤트가 유실될 수 있다. 
  - 메시지 큐(SQS/SNS, Kafka)를 이벤트 버스로 두고 WebSocket/Server-Sent Events로 최종 전파하는 구조가 일반적이다.
- **근거 자료**:
  - [Designing Uber: Geospatial Indexing, WebSockets, and Distributed Locks](https://dev.to/ganesh_parella/designing-uber-geospatial-indexing-websockets-and-distributed-locks-4mhb)
  - [The Power of MQTT and Confluent in Fleet Management](https://www.confluent.io/blog/fleet-management/)
