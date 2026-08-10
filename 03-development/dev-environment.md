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

> ⚠️ 위 표는 단일 프로젝트 기준으로 작성됐다. 지금은 [서비스가 3개](../02-design/architecture.md)이므로 이름·아티팩트·패키지명은 서비스별로 나뉜다. 나머지(Spring Boot 4.1.0 / Java 17 / Gradle Kotlin DSL / JDK Corretto 21.0.12 / Jar / Properties)는 **3개 서비스 모두 동일하게** 맞춘다.

### 서비스별 명명 규칙

**아티팩트명 = 레포명**, **패키지명 = `com.example.` + 아티팩트명에서 하이픈 제거.**

Spring Initializr는 Group과 Artifact를 입력하면 Package name을 이 규칙대로 자동으로 채운다. **Package name 칸은 직접 수정하지 않는다.**

| 서비스 | 레포 = Name = Artifact | Package | 메인 클래스 |
|---|---|---|---|
| 정산 | `driver-settlement-system` | `com.example.driversettlementsystem` | `DriverSettlementSystemApplication` |
| 결제 | 🚧 미정 | 레포명 확정 시 위 규칙대로 | |
| 원장 | 🚧 미정 | 레포명 확정 시 위 규칙대로 | |

> 레포명을 바꿔도 `settings.gradle.kts`의 `rootProject.name`은 따라가지 않는다. 함께 고쳐야 산출물 jar 이름이 맞는다.

---

## 🗄️ 데이터베이스 — Supabase PostgreSQL

**인스턴스 1개를 3개 서비스가 공유한다.** 테이블 소유권과 접근 규칙은 [erd.md](../02-design/erd.md)와 [service-contracts.md §0](../02-design/service-contracts.md#0-테이블-직접-접근-금지)을 먼저 읽는다.

### 접속 정보

**연결 문자열·비밀번호·API 키를 이 문서나 코드에 적지 않는다.** 이 레포가 private이어도 마찬가지다 — 커밋된 비밀정보는 이력에 영구히 남는다.

| 환경변수 | 내용 | 획득 경로 |
|---|---|---|
| `SPRING_DATASOURCE_URL` | JDBC 연결 문자열 | Supabase 대시보드 → Project Settings → Database |
| `SPRING_DATASOURCE_USERNAME` | DB 사용자 | 위와 동일 |
| `SPRING_DATASOURCE_PASSWORD` | DB 비밀번호 | 위와 동일 |

- 로컬에서는 IntelliJ 실행 구성의 환경변수 또는 `.env`(**반드시 `.gitignore`에 추가**)로 주입한다.
- `application.properties`에는 `${SPRING_DATASOURCE_URL}` 형태의 참조만 둔다.
- 🚧 실제 접속 정보 공유 경로를 정해야 한다(팀 DM / 비밀번호 관리 도구). 문서에는 적지 않는다.

### 커넥션 풀 ⚠️

서비스가 3개라 **HikariCP 풀도 3개가 열린다.** 서버당 기본 풀 크기가 10이면 최대 30개 연결이 Supabase 하나로 몰린다. Supabase는 플랜별 동시 연결 한도가 있어 넘기면 연결 거부가 난다.

- 🚧 **서버당 풀 크기 상한**을 정해야 한다 (`spring.datasource.hikari.maximum-pool-size`)
- 🚧 **Supabase 연결 풀러(Supavisor) 경유 여부** — 경유하면 연결 수를 아낄 수 있으나, transaction 모드에서는 prepared statement가 제한돼 JDBC 설정 조정이 필요하다. 직접 연결로 갈지 풀러로 갈지 정한 뒤 3개 서비스가 **같은 방식**을 쓴다

### 스키마 마이그레이션 ⚠️

DB가 공유라서 **3명이 각자 스키마를 바꾸면 서로의 로컬을 깨뜨린다.** 절차를 정해야 한다.

- 🚧 **도구**: Flyway / Supabase 마이그레이션 / 대시보드에서 수동 SQL
  - 서비스가 3개인데 Flyway를 각각 쓰면 버전 번호가 충돌한다. 서비스별 스키마를 분리하거나([erd.md §3](../02-design/erd.md#3-소유권을-무엇으로-표현할까)), 마이그레이션을 한 곳에서 관리해야 한다
- 🚧 **절차**: 스키마를 바꾸기 전에 `schema` 라벨로 이슈를 열고 나머지 2명을 멘션한다 ([github-workflow.md §2](../00-team/github-workflow.md#2-이슈는-어디에-내나))
- 🚧 **적용 순서**: 누가 언제 원격 DB에 적용하는지

### 로컬 개발용 DB ⚠️

🚧 공유 인스턴스 하나를 셋이 같이 쓰면 **서로의 테스트 데이터가 섞이고**, 한 명이 테이블을 비우면 다른 사람의 테스트가 깨진다. 선택지:

- 각자 로컬 PostgreSQL(Docker) + 공유 Supabase는 통합 테스트·데모용으로만
- Supabase 프로젝트를 개발용/데모용 2개로 분리
- 공유 하나로 쓰되 테스트 데이터 접두사 규칙으로 구분

---

## 관련 문서

- 전체 아키텍처: [architecture.md](../02-design/architecture.md)
- 테이블 소유권: [erd.md](../02-design/erd.md)
- 프로젝트 개요: [overview.md](../01-planning/overview.md)
