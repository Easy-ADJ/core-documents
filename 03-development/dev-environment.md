# 🛠️ 개발 환경

## 프로젝트 셋팅

IDE는 **IntelliJ IDEA**, 프로젝트는 [start.spring.io](https://start.spring.io)에서 생성한다.

| 설정 요소 | 설정값 |
|---|---|
| Spring Boot | 4.1.0 |
| Java | 17 |
| JDK | Amazon Corretto 21.0.12 |
| Gradle | Kotlin DSL 8.7 |
| 그룹명 | `com.example` |
| 이름 · 아티팩트 | 레포지토리명 |
| 패키지명 | `com.example.` + 아티팩트명(하이픈 제거) |
| 패키징 | Jar |
| 구성 | Properties |

이름·아티팩트·패키지명만 서비스별로 다르고, **나머지는 3개 서비스가 전부 동일하다.**

### 서비스별 명명

| 서비스 | 레포 = Name = Artifact | Package | 메인 클래스 |
|---|---|---|---|
| 결제 | `driver-payment-system` | `com.example.driverpaymentsystem` | `DriverPaymentSystemApplication` |
| 원장 | `driver-ledger-system` | `com.example.driverledgersystem` | `DriverLedgerSystemApplication` |
| 정산 | `driver-settlement-system` | `com.example.driversettlementsystem` | `DriverSettlementSystemApplication` |

- Spring Initializr가 Group·Artifact로 Package name을 자동으로 채운다. **직접 수정하지 않는다.**
- 레포명을 바꿔도 `settings.gradle.kts`의 `rootProject.name`은 따라가지 않는다. 함께 고쳐야 jar 이름이 맞는다.
- 가상 스레드를 켠다 — `spring.threads.virtual.enabled=true`

> 🚧 **위 표는 규칙이고, 현재 실제 코드는 결제·원장 두 곳에서 규칙과 다르다.** (2026-08-14 확인)
>
> | 서비스 | 실제 패키지 | 실제 메인 클래스 |
> |---|---|---|
> | 결제 | `com.example.eazyadj` | — |
> | 원장 | `com.example.easyadj` | `EasyadjApplication` |
> | 정산 | `com.example.driversettlementsystem` ✅ | `DriverSettlementSystemApplication` ✅ |
>
> 결제는 `eazy`, 원장은 `easy`로 **둘이 서로도 다르다.** 지금은 서비스 간 호출이 REST뿐이라 당장 깨지는 것은 없지만, 나중에 공통 모듈을 만들거나 로그·모니터링에서 패키지 prefix로 서비스를 구분할 때 걸린다.
>
> **패키지명을 규칙에 맞출지, 표를 실제에 맞출지는 각 소유자(김주엽·이치헌)의 결정이다.** 정하기 전까지 이 표를 근거로 남의 클래스 경로를 추측하지 않는다.

---

## 데이터베이스

**Supabase** PostgreSQL 프로젝트 4개를 쓴다. 각 서버는 자기 DB와 로그인 DB, 두 곳에 접속한다.
테이블 소유권과 접근 규칙은 [erd.md](../02-design/erd.md)와 [service-contracts.md §0](../02-design/service-contracts.md#0-테이블-직접-접근-금지)을 먼저 읽는다.

- JDBC는 **Supavisor session 모드**로 붙는다. transaction 모드는 prepared statement가 제한돼 URL에 `prepareThreshold=0` 같은 설정이 붙고, 빠뜨리면 간헐적 에러로 드러난다.
- 스키마는 **Flyway**로 관리한다. `V1__init.sql` 형태로 레포에 남긴다.

> ⚠️ Supabase는 잠정 선택이다. AWS 계정 지급이 지연돼 택했다.
> 나중에 옮길 수 있으므로 **제품에 종속되는 코드를 쓰지 않는다** — 접속 정보는 환경변수, 스키마는 Flyway.

### 접속 정보

**연결 문자열·비밀번호·API 키를 문서나 코드에 적지 않는다.** private 레포여도 마찬가지다 — 커밋된 비밀정보는 이력에 영구히 남는다.

서버마다 DataSource가 2개이므로 환경변수도 두 벌이다.

| 환경변수 | 내용 | 획득 경로 |
|---|---|---|
| `SPRING_DATASOURCE_URL` | 자기 DB 연결 문자열 | 자기 Supabase 프로젝트 |
| `SPRING_DATASOURCE_USERNAME` | DB 사용자 | 위와 동일 |
| `SPRING_DATASOURCE_PASSWORD` | DB 비밀번호 | 위와 동일 |
| `AUTH_DATASOURCE_URL` | 로그인 DB 연결 문자열 | 허진수 |
| `AUTH_DATASOURCE_USERNAME` | 로그인 DB 사용자 (읽기 전용 계정) | 위와 동일 |
| `AUTH_DATASOURCE_PASSWORD` | 로그인 DB 비밀번호 | 위와 동일 |
| `LEDGER_API_BASE_URL` | 원장 서버 주소 | 이치헌 |
| `PAYMENT_API_BASE_URL` | 결제 서버 주소 (정산만) | 김주엽 |

- 로컬에서는 IntelliJ 실행 구성의 환경변수 또는 `.env`(**반드시 `.gitignore`**)로 주입한다.
- `application.properties`에는 `${SPRING_DATASOURCE_URL}` 형태의 참조만 둔다.
- 로그인 DB는 **서버별 읽기 전용 계정**으로 붙는다 — `payment_ro` · `ledger_ro` · `settlement_ro`. 한 곳이 유출돼도 그 계정만 회수하면 된다.
- 남의 업무 DB 접속 정보는 요청하지도 공유하지도 않는다.

### 다중 DataSource 설정

Spring Boot는 DataSource가 2개면 **자동 설정이 걸리지 않는다.** 세 서버 모두 아래 형태로 통일한다.

```java
/**
 * 자기 서비스 DB — 기본 DataSource.
 */
@Configuration
@EnableJpaRepositories(
        basePackages = "com.example.<서비스>.repository",
        entityManagerFactoryRef = "primaryEntityManagerFactory",
        transactionManagerRef = "primaryTransactionManager"
)
public class PrimaryDataSourceConfig
{
    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource")
    public DataSourceProperties primaryDataSourceProperties()
    {
        return new DataSourceProperties();
    }

    @Bean
    @Primary
    public DataSource primaryDataSource()
    {
        return primaryDataSourceProperties()
                .initializeDataSourceBuilder()
                .build();
    }
}

/**
 * 로그인 DB — 기사 정보 조회 전용.
 */
@Configuration
@EnableJpaRepositories(
        basePackages = "com.example.<서비스>.auth.repository",
        entityManagerFactoryRef = "authEntityManagerFactory",
        transactionManagerRef = "authTransactionManager"
)
public class AuthDataSourceConfig
{
    @Bean
    @ConfigurationProperties("auth.datasource")
    public DataSourceProperties authDataSourceProperties()
    {
        return new DataSourceProperties();
    }

    @Bean
    public DataSource authDataSource()
    {
        return authDataSourceProperties()
                .initializeDataSourceBuilder()
                .build();
    }
}
```

```properties
spring.datasource.url=${SPRING_DATASOURCE_URL}
spring.datasource.username=${SPRING_DATASOURCE_USERNAME}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD}

auth.datasource.url=${AUTH_DATASOURCE_URL}
auth.datasource.username=${AUTH_DATASOURCE_USERNAME}
auth.datasource.password=${AUTH_DATASOURCE_PASSWORD}
auth.datasource.hikari.maximum-pool-size=3
```

- `EntityManagerFactory`·`TransactionManager` Bean은 위 `DataSource`를 받아 각 설정 클래스에 함께 등록한다.
- **패키지를 `auth.repository`로 분리하는 것이 중요하다.** 두 `@EnableJpaRepositories`의 `basePackages`가 겹치면 어느 DataSource로 갈지 모호해져 기동이 실패한다.
- ⚠️ **두 DB에 걸친 트랜잭션과 JOIN은 불가능하다.** 각각 조회해 애플리케이션에서 합친다.

### 커넥션 풀

업무 DB는 서버 하나가 자기 것만 쓰므로 경합이 없다. HikariCP 기본값(10)을 그대로 쓴다.

⚠️ **로그인 DB만 다르다.** 세 서버 + 클라이언트가 한 인스턴스에 붙어 경합이 생긴다. 기본값이면 서버 3대만으로 30 커넥션이므로 **3으로 고정**한다. 기사 정보 조회는 빈도가 낮다.

### 스키마 마이그레이션

업무 DB는 나뉘어 서로의 스키마를 깨뜨릴 수 없다. 각자 독립적으로 관리하며 Flyway 버전 번호도 충돌하지 않는다.

⚠️ **로그인 DB는 예외다.** `DRIVER_ACCOUNTS` 컬럼 하나만 바꿔도 세 서버 쿼리가 한꺼번에 깨진다. 마이그레이션은 소유자(허진수)만 수행하고, 다른 서버는 자기 마이그레이션 대상에 로그인 DB를 넣지 않는다.

스키마를 바꾸기 전에 `schema` 라벨로 이슈를 연다. ([github-workflow.md](../00-team/github-workflow.md))

### 로컬 개발용 DB

**세 사람 모두 같은 Supabase 프로젝트를 본다.** 로컬 Docker를 따로 띄우지 않는다 — DB가 4개라 구성·동기화 부담이 크고, 통합 테스트를 바로 할 수 있는 편익이 더 크다.

- 업무 DB(결제·원장·정산)는 각자 자유롭게 비우고 채워도 된다. 서로 격리돼 있다.
- **데모용 기사 계정은 고정해두고 누구도 지우거나 수정하지 않는다.** 테스트가 필요하면 새 계정을 만든다.
- ⚠️ 데모 직전에는 아무도 데이터를 지우지 않는다. 원격 하나를 공유하므로 한 사람의 초기화가 곧 시연 실패다.

---

## 클라이언트

| 요소 | 선택 |
|---|---|
| 언어 | TypeScript |
| 프레임워크 | React · Next.js |
| 컴포넌트 | shadcn/ui |
| 스타일 | Tailwind CSS |
| 배포 | Vercel |
| 인증 | 로그인·회원가입을 Next.js에서 구현, 서버 호출 시 `Bearer Token` |

- 화면은 관리자용과 기사용을 따로 만든다.
- 로그인 DB에는 **Supabase anon key + RLS**로 접근한다. DB 자격증명을 클라이언트에 두지 않는다.
- 레포: [dashboard-system](https://github.com/Easy-ADJ/dashboard-system)

---

## 관련 문서

- 전체 구성: [architecture.md](../02-design/architecture.md)
- 테이블 구조: [erd.md](../02-design/erd.md)
