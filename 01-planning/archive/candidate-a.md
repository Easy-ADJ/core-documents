# 실시간 차량 위치 관제 대시보드 — 상세 기획 (후보 A)

> 📦 **보관 문서 — 현행 아님.** 미채택 후보의 기획 초안입니다. 현재 기획은 [service-spec.md](../service-spec.md), 현행 아키텍처는 [architecture.md](../../02-design/architecture.md)를 참조하세요.

> [project-candidates.md](./project-candidates.md)의 **후보 A(실시간 차량 위치 관제 대시보드)** — 문제 1 "대규모 동시 위치 업데이트 처리" 대응 — 를 실제 개발에 들어갈 수 있는 수준으로 구체화한 문서다. 기술 스택은 Java 17+ / **Spring Boot 4** / **AWS** 기준.

---

## 1. 문제 재정의

모빌리티 플랫폼은 수십~수천 대의 차량이 수 초 단위로 GPS 좌표를 전송하고, 이를 실시간으로 저장·조회·전파해야 한다. 이때 실무적으로 부딪히는 어려움은 세 가지로 정리된다.

1. **대량 동시 쓰기**: 모든 차량이 거의 동시에 위치를 갱신하므로, 서버는 초당 수백~수만 건의 쓰기를 처리해야 한다.
2. **저지연 반경 검색**: "지금 이 지점 반경 3km 안에 있는 차량"을 실시간으로 조회해야 하는데, 일반 RDB의 좌표 조회는 이런 질의에 느리다.
3. **신호 품질 문제**: 배터리 절약을 위한 샘플링 주기 조절, 터널·건물 밀집 지역에서의 신호 유실/끊김 등으로 "차량이 화면에서 사라지거나 위치가 순간이동하는" 현상이 생긴다.

Uber는 이 문제를 지속적인 연결(WebSocket)로 위치를 수신해 Redis 기반 지오스페이셜 캐시에 저장하고, 위치 갱신을 200ms 이내에 전파하는 구조로 해결한다. ([How Uber Scales Real-Time Location Tracking for Millions of Users](https://medium.com/@wahab9766/how-uber-scales-real-time-location-tracking-for-millions-of-users-ff5c43ab6632))

---

## 2. 시나리오

> 당신은 지역 소형 화물 운송사를 위한 관제 플랫폼 "D-Fleet"의 백엔드 개발팀에 합류했다.
>
> D-Fleet은 차량에 부착된 단말이 5초마다 위치를 서버로 전송하고, 관제 대시보드에서 관리자가 모든 차량의 실시간 위치를 지도에서 확인한다.
>
> 시범 운영 중 두 가지 문제가 보고됐다.
> - 차량이 30대를 넘어가자 대시보드 갱신이 눈에 띄게 느려졌고, 일부 차량 아이콘이 몇 초씩 "멈춰 있다가 갑자기 점프"하는 현상이 나타났다.
> - 배차 담당자가 "여기서 가장 가까운 차량 5대"를 찾으려 할 때마다 응답이 2~3초씩 걸려, 배차 업무에 병목이 생겼다.
>
> 팀 리드가 말했다.
> "이번 스프린트에서는 두 가지를 증명해야 해요. 첫째, 차량 수가 늘어나도 위치 갱신이 실시간으로 부드럽게 보일 것. 둘째, 반경 검색이 즉시 응답할 것."

---

## 3. 핵심 기능 명세

| 기능 | 설명 |
|---|---|
| 위치 업데이트 수신 API | `POST /api/vehicles/{vehicleId}/locations` — 차량 단말(또는 시뮬레이터)이 좌표·속도·타임스탬프를 전송 |
| 실시간 좌표 저장 | 최신 좌표를 **Redis(ElastiCache) Geospatial 인덱스**에 저장(`GEOADD`) — 갱신 시마다 덮어쓰기 |
| 반경 검색 API | `GET /api/vehicles/nearby?lat=&lng=&radiusKm=` — `GEOSEARCH`로 반경 내 차량을 거리순으로 조회 |
| 실시간 브로드캐스트 | WebSocket/STOMP 구독 클라이언트(관제 대시보드)에게 위치 변경 이벤트를 즉시 push |
| 이동 경로 히스토리 | 좌표를 RDS에도 비동기 적재해, 특정 차량의 지난 N시간 이동 경로를 조회 (`GET /api/vehicles/{vehicleId}/history`) |
| 오프라인 감지 | Redis 키에 TTL(예: 30초)을 설정, TTL 만료 시 "연결 끊김"으로 판단해 대시보드에 별도 표시 — 위치가 순간이동하는 것처럼 보이는 문제를 "끊김"으로 명확히 구분 |
| 부하 시뮬레이터 | 다수의 가상 차량이 동시에 위치를 계속 전송하는 테스트 클라이언트(별도 콘솔 앱/스크립트) — 실제 시연 및 성능 검증용 |

---

## 4. 아키텍처

### 4.1 시스템 구성도 (텍스트)

```
[차량 단말 시뮬레이터 다수] ──(HTTP POST, 5초 주기)──▶ [Spring Boot 4 API 서버]
                                                          │
                                        ┌─────────────────┼─────────────────┐
                                        ▼                                   ▼
                        [Amazon ElastiCache for Redis]         [Amazon RDS]
                        - GEOADD (최신 좌표, TTL)                - 위치 이력 테이블(비동기 적재)
                        - GEOSEARCH (반경 검색)
                                        │
                                        ▼
                        [Spring WebSocket/STOMP 브로커]
                                        │
                                        ▼
                        [관제 대시보드 클라이언트 (지도 뷰)]
```

### 4.2 데이터 모델 개요

- **Vehicle**(차량) — id, name, type
- **VehicleLocationHistory**(이력, RDS) — id, vehicleId, lat, lng, speed, recordedAt
- Redis: `GEO` 키 하나(`vehicle:locations`)에 모든 차량의 최신 좌표를 멤버로 저장, 별도 Hash에 최근 갱신 시각·속도 등 메타데이터 저장

### 4.3 기술 선택과 근거

| 요소 | 선택 | 근거 |
|---|---|---|
| 최신 좌표 저장·조회 | Redis Geospatial(`GEOADD`/`GEOSEARCH`) | 인메모리 처리로 지리공간 질의가 밀리초 이하 지연으로 동작해 반경 검색 요구사항에 적합 |
| 실시간 전파 | Spring WebSocket/STOMP | 폴링 대비 지연이 적고, 다수 클라이언트에 동시 브로드캐스트 가능 |
| 대량 동시 쓰기 처리 | Spring Boot 4 가상 스레드 | 위치 업데이트 API는 Redis/DB I/O 대기가 대부분이라, 가상 스레드로 별도 리액티브 전환 없이 동시 처리량 확보 |
| 오프라인 판정 | Redis 키 TTL | 별도 하트비트 관리 로직 없이 TTL 만료만으로 "연결 끊김" 판정 가능 |
| 이력 저장 | RDS (비동기) | 실시간 경로는 Redis, 장기 이력 조회는 RDB로 역할을 분리해 실시간 경로에 부하를 주지 않음 |

- 참고: [Redis Geo Commands Tutorial](https://redis.io/tutorials/howtos/solutions/geo/getting-started/)
- 참고: [Redis geospatial 공식 문서](https://redis.io/docs/latest/develop/data-types/geospatial/)

### 4.4 동시성·성능 전략

- 위치 업데이트 API는 상태를 갖지 않는(stateless) 단순 쓰기이므로 별도 락 없이 수평 확장 가능
- Redis `GEOADD`는 원자적 연산이라 동시 갱신 시에도 데이터 손상 없음
- 이력 적재는 API 응답 흐름과 분리(비동기 큐 또는 `@Async`)해, 대량 쓰기가 반경 검색 응답 속도에 영향을 주지 않도록 함

---

## 5. 개발 로드맵 (실제 개발 가능 기간 3~4주 기준)

> 프로그램 일정 기준: 6주차(07.27~07.31) 기획 마무리 → **7주차(08.03~08.07) / 8주차(08.10~08.14) / 9주차(08.17~08.21)** 3주간 개발 → 9주차 팀밋업(08.18) 중간 점검 → 10주차 성과교류회(08.25) 최종 발표.

### 7주차 (08.03~08.07) — 위치 수신 및 저장

- Day 1~2: `Vehicle`, `VehicleLocationHistory` 도메인 설계, ElastiCache Redis 클러스터 프로비저닝
- Day 3~4: 위치 업데이트 API 구현, Redis `GEOADD` 연동, TTL 설정
- Day 5: 반경 검색 API(`GEOSEARCH`) 구현 및 단위 테스트
- **완료 기준**: 임의의 좌표를 넣었을 때 반경 검색 API가 올바른 거리순 결과를 반환

### 8주차 (08.10~08.14) — 실시간 전파 및 대시보드 연동

- Day 1~2: WebSocket/STOMP 브로커 설정, 위치 변경 이벤트 구독/발행 구조 구현
- Day 3~4: 간단한 지도 대시보드(프론트엔드 또는 정적 페이지)에서 실시간 위치 마커 갱신 확인
- Day 5: 오프라인 감지(TTL 만료) 로직 및 대시보드 상 "연결 끊김" 표시
- **완료 기준**: 위치 업데이트 후 1초 이내에 대시보드에서 마커가 이동, 업데이트가 끊기면 일정 시간 후 "오프라인"으로 표시

### 9주차 (08.17~08.21) — 부하 테스트, 이력 조회, 발표 준비

- Day 1~2: 다수 가상 차량 부하 시뮬레이터 작성, 동시 30~100대 시뮬레이션
- Day 3: 이동 경로 히스토리 API 구현(RDS 비동기 적재 포함)
- Day 4: AWS 배포(EC2/Elastic Beanstalk + ElastiCache + RDS), 성능 측정 결과 정리
- Day 5(팀밋업 08.18 전후): 발표자료·데모 시나리오(부하 시뮬레이터 실행 → 대시보드 실시간 반응 시연) 준비
- **완료 기준**: 다수 차량 동시 시뮬레이션 상황에서도 반경 검색 응답 시간과 대시보드 갱신 지연을 수치로 제시 가능

---

## 6. 참고 자료

- [How Uber Scales Real-Time Location Tracking for Millions of Users](https://medium.com/@wahab9766/how-uber-scales-real-time-location-tracking-for-millions-of-users-ff5c43ab6632)
- [How Uber's Real-Time Location Tracking System Works at Scale](https://www.bnxt.ai/blog/how-ubers-real-time-location-tracking-system-works-at-scale)
- [카카오 T 택시의 배차 시스템 - 카카오모빌리티](https://www.kakaomobility.com/contents/taxi-dispatch)
- [Redis Geo Commands Tutorial: Location-Based Queries and Search](https://redis.io/tutorials/howtos/solutions/geo/getting-started/)
- [Redis geospatial - 공식 문서](https://redis.io/docs/latest/develop/data-types/geospatial/)
- [How to Use GEOSEARCH in Redis for Flexible Geo Queries](https://oneuptime.com/blog/post/2026-03-31-redis-geosearch-flexible-geo-queries/view)
- [Designing Uber: Geospatial Indexing, WebSockets, and Distributed Locks](https://dev.to/ganesh_parella/designing-uber-geospatial-indexing-websockets-and-distributed-locks-4mhb)
