# 📚 EasyADJ — 프로젝트 문서

모빌리티 정산 서비스 <u>**EasyADJ**</u>의 프로젝트 코어 문서 관리용 레포지토리입니다.<br/>
승차 요금을 <u>**중복 없이 한 번만**</u> 청구하고, 기사에게 지급할 정산 금액을 <u>**원장만 보고 설명할 수 있게**</u> 만드는 것이 목표입니다.

## ⚙️ 시스템 구성

결제·원장·정산 <u>**3개 서버**</u>를 담당자별로 나눠 개발하고 API를 통해 서로 통신합니다.<br/>
DB는 **AWS Aurora** 혹은 **Supabase** 서비스를 활용하여 <u>**PostgreSQL (ORDBMS)**</u>을 사용할 예정이며,<br/>
<u>**시스템마다 인스턴스를 하나씩**</u> 둡니다. (2026-08-11 회의 결정)

| 서비스 | 담당 | 레포 |
|---|---|---|
| 결제 (payment) | 김주엽 | [driver-payment-system](https://github.com/Easy-ADJ/driver-payment-system) |
| 원장 (ledger) | 이치헌 | [driver-ledger-system](https://github.com/Easy-ADJ/driver-ledger-system) |
| 정산 (settlement) | 허진수 | [driver-settlement-system](https://github.com/Easy-ADJ/driver-settlement-system) |
| 코어 문서 | 전원 | [core-documents](https://github.com/Easy-ADJ/core-documents) |

> DB가 나뉘어 <u>**남의 테이블에는 접속할 수 없습니다.**</u> 경계를 넘는 데이터는 전부 API를 거칩니다.<br/>
> 자세한 규칙은 [service-contracts의 §0](./02-design/service-contracts.md#0-테이블-직접-접근-금지) 참고하세요.

---

## 📖 문서 목차

### 👥 [00-team/](./00-team/) — 누가, 어떤 규칙으로

- [roles.md](./00-team/roles.md) — 담당자별 서비스·레포·소유 문서·소유 테이블
- [links.md](./00-team/links.md) — `GitHub` / `AWS Aurora` 혹은 `Supabase` / `Trello` / `ERDCloud` / `Draw.io`
- [github-workflow.md](./00-team/github-workflow.md) — <u>**레포 4개에서 이슈를 어디에 낼지**</u>, 라벨, Projects 보드
- [code-convention.md](./00-team/code-convention.md) — 네이밍, 포매터, 코드 스타일, IntelliJ 설정
- [git-convention.md](./00-team/git-convention.md) — 커밋 메시지, 커밋 단위, 브랜치 플로우

### 🎯 [01-planning/](./01-planning/) — 무엇을 왜 만드는가

- **[service-spec.md](./01-planning/service-spec.md) — <u>확정 기획서</u>** (기능 명세, 개발 로드맵)
- [overview.md](./01-planning/overview.md) — 프로젝트 개요, 과정 정보, 기술 스택
- [archive/](./01-planning/archive/) — 후보 탐색 기록 (현행 아님)

### 📐 [02-design/](./02-design/) — 어떻게 만들 것인가

- [architecture.md](./02-design/architecture.md) — <u>**아키텍처 단일 출처.**</u> 3서버·3DB 구성, 서비스 경계, 설계 논점
- [service-contracts.md](./02-design/service-contracts.md) — 서버 간 호출 계약, 테이블 접근 규칙, 계약 변경 절차
- [erd.md](./02-design/erd.md) — 테이블 구조와 소유권
- [services/](./02-design/services/) — 서버별 API 명세 ([payment](./02-design/services/payment.md) / [ledger](./02-design/services/ledger.md) / [settlement](./02-design/services/settlement.md))

### 🛠️ [03-development/](./03-development/) — 만들면서 생긴 것

- [dev-environment.md](./03-development/dev-environment.md) — IntelliJ·Gradle 셋팅, <u>**AWS Aurora / Supabase 연결·마이그레이션**</u>
- [logs/](./03-development/logs/) — 개발 일지·트러블슈팅 ([템플릿](./03-development/logs/_template.md))

### 📝 [04-meetings/](./04-meetings/) — 회의록 ([템플릿](./04-meetings/_template.md))

### 📅 [05-milestones/](./05-milestones/) — 일정과 발표

- [schedule.md](./05-milestones/schedule.md) — 전반적인 스케줄

---

## ✍️ 문서 관리 규칙

### 🗂️ 이름

- 폴더는 `NN-name`, 파일은 `영문 kebab-case` ([추가적 서비스 제약 참고](./02-design/service-contracts.md))
- 시계열 문서는 `YYYY-MM-DD-topic.md`로, 개발 로그는 서버명을 넣습니다.
	예) `2026-08-11-ledger-balance-api.md`
- 문서 제목(H1)은 이모지 1개 + 한글

### 🔒 수정 권한

| 문서 | 수정할 수 있는 사람 |
|---|---|
| `02-design/services/*.md` | <u>**해당 서버 담당자만**</u> |
| `02-design/service-contracts.md`, `erd.md`, `architecture.md` | 합의 후 [계약 변경 절차](./02-design/service-contracts.md#3-계약-변경-절차) 수행 |
| 그 외 | 누구나 |

### 지켜야 할 것

- <u>**연결 문자열·API 키·비밀번호를 문서에 적지 않습니다.**</u> (public/private 레포 모두 포함)
- 이미지는 해당 문서와 같은 폴더의 `images/`에 둡니다.
- 유효하지 않게 된 문서는 <u>**삭제하지 말고**</u> `archive/`로 옮기고 상단에 "현행 아님" 배너를 답니다.
- 설계 결정은 회의록과 <u>**해당 설계 문서에 반영**</u>합니다.
- 커밋 메시지는 `docs:` prefix ([Git 컨벤션 참고](./00-team/git-convention.md))
