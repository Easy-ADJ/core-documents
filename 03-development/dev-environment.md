# 🛠️ 개발 환경

## 🏞️ Code Editor : **Intellij IDEA**

## 💻 프로젝트 셋팅

| 설정 요소 | 설정값 | 비고 |
| :---: | :---: | :--- |
| Spring Boot 4 | 4.1.0 | - |
| 서버 URL | https://start.spring.io | - |
| 이름 | { System 레포지토리명 } | - |
| 프로젝트 위치 | 로컬 환경에서 개인적으로 설정 | - |
| Java | 17 | - |
| Gradle | Kotlin DSL | 8.7 |
| 그룹명 | com.example | - |
| 아티팩트 | { System 레포지토리명 } | - |
| 패키지명 | com.example.{ System 레포지토리명(하이폰 제외) } | - |
| JDK | Amazon Corretto | 21.0.12 |
| 패키지 생성 | Jar | - |
| 구성 | Properties | - |

> 이름·아티팩트·패키지명은 [서비스가 3개](../02-design/architecture.md)이므로 서비스별로 나뉜다.<br/>
> 나머지(Spring Boot 4.1.0 / Java 17 / Gradle Kotlin DSL / JDK Corretto 21.0.12 / Jar / Properties)는 <u>**3개 서비스 모두 동일하게**</u> 맞춘다.

### 서비스별 명명 규칙

<u>**아티팩트명 = 레포명**</u>, <u>**패키지명 = `com.example.` + 아티팩트명에서 하이픈 제거.**</u>

Spring Initializr는 Group과 Artifact를 입력하면 Package name을 이 규칙대로 자동으로 채운다.<br/>
<u>**Package name 칸은 직접 수정하지 않는다.**</u>

| 서비스 | 레포 = Name = Artifact | Package | 메인 클래스 |
|---|---|---|---|
| 결제 | `driver-payment-system` | `com.example.driverpaymentsystem` | `DriverPaymentSystemApplication` |
| 원장 | `ledger-system` | `com.example.ledgersystem` | `LedgerSystemApplication` |
| 정산 | `driver-settlement-system` | `com.example.driversettlementsystem` | `DriverSettlementSystemApplication` |

> 레포명을 바꿔도 `settings.gradle.kts`의 `rootProject.name`은 따라가지 않는다. 함께 고쳐야 산출물 jar 이름이 맞는다.

---

## 🗄️ 데이터베이스 — 시스템별 PostgreSQL

<u>**시스템마다 DB 인스턴스를 하나씩 둔다.**</u> 각 서버는 자기 DB에만 접속한다.<br/>
테이블 소유권과 접근 규칙은 [ERD 문서](../02-design/erd.md)와 [service-contracts.md §0](../02-design/service-contracts.md#0-테이블-직접-접근-금지)을 먼저 읽는다.

- 🚧 DB 제품 미확정 — **AWS Aurora** 혹은 **Supabase**. 어느 쪽이든 <u>**3개 시스템이 같은 제품**</u>을 쓴다. ([architecture.md §5](../02-design/architecture.md#5-인프라))

### 접속 정보

<u>**연결 문자열·비밀번호·API 키를 이 문서나 코드에 적지 않는다.**</u> 이 레포가 private이어도 마찬가지다 — 커밋된 비밀정보는 이력에 영구히 남는다.

| 환경변수 | 내용 | 획득 경로 |
|---|---|---|
| `SPRING_DATASOURCE_URL` | JDBC 연결 문자열 | 자기 DB의 콘솔·대시보드 |
| `SPRING_DATASOURCE_USERNAME` | DB 사용자 | 위와 동일 |
| `SPRING_DATASOURCE_PASSWORD` | DB 비밀번호 | 위와 동일 |

- 각자 <u>**자기 서비스 DB의 접속 정보만**</u> 갖는다. 남의 DB 접속 정보를 요청하지도, 공유하지도 않는다.
- 로컬에서는 IntelliJ 실행 구성의 환경변수 또는 `.env`(<u>**반드시 `.gitignore`에 추가**</u>)로 주입한다.
- `application.properties`에는 `${SPRING_DATASOURCE_URL}` 형태의 참조만 둔다.
- 🚧 서버 간 호출 대상 주소도 환경변수로 주입한다. 변수 이름 규칙을 정해야 한다. (예: `LEDGER_API_BASE_URL`)

### 커넥션 풀

DB가 나뉘어 <u>**서버 하나가 자기 DB만 쓰므로 풀 경합이 없다.**</u> HikariCP 기본값(풀 크기 10)을 그대로 써도 된다.

- 부하 테스트에서 부족하면 그때 `spring.datasource.hikari.maximum-pool-size`를 올린다. 미리 튜닝하지 않는다.
- 🚧 Supabase로 확정되는 경우에 한해, 연결 풀러(Supavisor) 경유 여부를 정한다. transaction 모드에서는 prepared statement가 제한돼 JDBC 설정 조정이 필요하다.

### 스키마 마이그레이션

DB가 나뉘어 <u>**서로의 스키마를 깨뜨릴 수 없다.**</u> 각자 자기 DB의 마이그레이션을 독립적으로 관리하며, Flyway 버전 번호도 충돌하지 않는다.

- 🚧 <u>**도구**</u>: Flyway / 콘솔에서 수동 SQL. DB가 독립이라 서비스마다 달라도 동작하지만, <u>**3개 서비스가 같은 도구**</u>를 쓰는 편이 문서·트러블슈팅이 쉽다.
- 스키마를 바꾸기 전에 `schema` 라벨로 이슈를 연다. ([github-workflow.md §2](../00-team/github-workflow.md#2-이슈는-어디에))<br/>
	DB는 나뉘었지만 스키마 변경은 <u>**그 테이블을 노출하는 API 응답을 바꾸므로**</u> 상대에게 영향이 간다.

### 로컬 개발용 DB

DB가 나뉘어 <u>**서로의 테스트 데이터가 섞이지 않는다.**</u> 각자 자기 DB를 자유롭게 비우고 채울 수 있다.

- 🚧 그래도 <u>**통합 테스트·데모용 환경을 따로 둘지**</u>는 정해야 한다. 데모 중에 누군가 자기 DB를 비우면 시연이 깨진다.
	- 로컬 PostgreSQL(Docker)로 개발 + 원격 DB는 통합 테스트·데모용으로만
	- 원격 DB 하나로 쓰되 데모 직전에 데이터를 고정

---

## 관련 문서

- 전체 아키텍처: [Architecture 문서](../02-design/architecture.md)
- 테이블 소유권: [ERD 문서](../02-design/erd.md)
- 프로젝트 개요: [overview.md](../01-planning/overview.md)
