# 🐙 GitHub 협업 방식

> 레포가 4개로 나뉜 상태에서 **이슈를 어디에 내고, 현황을 어디서 보는지**를 규정한다.
> 커밋 메시지·브랜치 규칙은 [git-convention.md](./git-convention.md) 참조.

---

## 1. 레포 구성

조직: [Easy-ADJ](https://github.com/Easy-ADJ)

| 레포 | 용도 | 담당 | 공개 |
|---|---|---|---|
| [core-documents](https://github.com/Easy-ADJ/core-documents) | 프로젝트 문서 전용 (이 레포) | 전원 | private |
| 🚧 결제 서버 | payment 서비스 | 김주엽 | 미생성 |
| 🚧 원장 서버 | ledger 서비스 | 이치헌 | 미생성 |
| [driver-settlement-system](https://github.com/Easy-ADJ/driver-settlement-system) | settlement 서비스 | 허진수 | public |

> **레포 이름**: 정산은 `driver-settlement-system`으로 확정됐다. 결제·원장은 🚧 미정 — 확정되면 이 표와 [links.md](./links.md), [dev-environment.md의 명명 규칙 표](../03-development/dev-environment.md#서비스별-명명-규칙)를 함께 갱신한다.
> 문서 레포가 `easyadj-docs`가 아니라 `core-documents`인 것처럼, 조직 이름(`Easy-ADJ`)이 이미 접두사 역할을 하므로 레포명에 `easyadj-`를 반복하지 않는다.

---

## 2. 이슈는 어디에 내나

**기본 원칙: 이슈는 각 서버 레포에 낸다.** 문서 레포에는 문서 관련 이슈만 낸다.

문제는 **서버 간 문제**다. "원장 잔액 API 응답이 정산에서 쓰기 어렵다"는 이슈는 원장 레포에 낼까, 정산 레포에 낼까?

> ### 📌 **고쳐야 하는 쪽(제공자) 레포에 내고, 소비자 담당자를 멘션한다.**

| 상황 | 어디에 | 멘션 |
|---|---|---|
| 원장 API 응답 형식을 바꿔야 함 | **원장 레포** | 그 API를 쓰는 결제·정산 담당자 |
| 정산 배치가 결제 API 호출에서 타임아웃 | **결제 레포** (고칠 쪽이 결제) | 정산 담당자 |
| 내 서버 내부 버그 | 내 레포 | 없음 |
| DB 스키마 변경 | **테이블 소유 서비스 레포** | **나머지 2명 전원** |
| 문서·구조 문제 | core-documents | 해당자 |

이렇게 하는 이유: 이슈가 **고칠 사람의 할 일 목록**에 있어야 처리된다. 요청한 쪽 레포에 쌓이면 정작 고칠 사람이 안 본다.

**DB 스키마 변경에 2명 전원을 멘션하는 이유**는 [DB가 하나](../02-design/erd.md)이기 때문이다. 한 명의 `ALTER TABLE`이 나머지 둘의 로컬 환경을 동시에 바꾼다.

---

## 3. 라벨

**3개 서버 레포에 동일한 라벨 세트를 만든다.** 라벨이 레포마다 다르면 통합 보드에서 필터가 안 걸린다.

| 라벨 | 용도 |
|---|---|
| `feat` | 기능 구현 |
| `bug` | 오류 수정 |
| `contract` | **서버 간 API 계약** 관련 — 혼자 결정하면 안 되는 것 |
| `schema` | **DB 스키마 변경** — 공유 DB라 전원 영향 |
| `blocked` | 다른 서버 작업을 기다리는 중 |
| `question` | 결정이 필요한 논의 |

`contract`와 `schema`가 붙은 이슈는 **[service-contracts.md](../02-design/service-contracts.md) 또는 [erd.md](../02-design/erd.md) 갱신까지 끝나야 close** 한다. 코드만 고치고 닫으면 문서가 뒤처진다.

---

## 4. 이슈 작성 시 포함할 것

별도 템플릿 파일을 두지 않는다. 대신 아래를 본문에 담는다.

- **무엇을**: 한 줄 요약. 제목에 `[서비스명]`을 붙이면 통합 보드에서 알아보기 쉽다
- **왜**: 어떤 상황에서 필요해졌는지
- **누가 영향받는지**: 다른 서버가 걸리면 담당자 멘션
- **막혀 있다면**: 무엇을 기다리는지 + 그 이슈 번호 (`blocked by Easy-ADJ/ledger-service#12`)

계약·스키마 이슈는 **변경 전/후를 나란히** 적는다. 말로만 설명하면 상대가 다르게 이해한다.

---

## 5. GitHub Projects 보드

**현황은 GitHub Projects 보드 한 곳에서만 본다.** 문서 레포에 현황판 파일을 두지 않는다 — 두 곳을 동기화하면 반드시 한쪽이 낡는다.

> 🚧 보드 URL 미정 — 생성 후 [links.md](./links.md)에 추가

**반드시 조직 레벨 Project로 만든다.** 레포별 Project는 그 레포 이슈만 담아서, 레포가 4개인 지금은 통합 뷰가 나오지 않는다.

- 조직 페이지 → Projects → New project → 3개 서버 레포를 모두 연결
- 컬럼(제안): `Backlog` / `Todo` / `In Progress` / `Blocked` / `Done`
- 이슈는 **만들 때 바로** 보드에 올린다. 나중에 몰아 올리면 보드가 현황을 못 보여준다
- 회의([04-meetings](../04-meetings/)) 시작은 이 보드를 함께 보는 것으로 한다

---

## 6. 브랜치와 PR

각 레포 공통으로 [git-convention.md](./git-convention.md)의 브랜치 플로우(`main` ← `develop` ← `feature/*`)를 따른다.

- `main` 병합은 PR로만, 팀원 합의 하에
- PR 본문에 관련 이슈 번호를 적는다 (`Closes #12`)
- **서버 간 계약을 바꾸는 PR은 상대 담당자를 리뷰어로 지정**한다. 계약 변경 절차는 [service-contracts.md §3](../02-design/service-contracts.md#3-계약-변경-절차)

---

## 관련 문서

- 커밋·브랜치 규칙: [git-convention.md](./git-convention.md)
- 팀 역할과 담당 서비스: [roles.md](./roles.md)
- 협업 도구 링크: [links.md](./links.md)
- 계약 변경 절차: [service-contracts.md](../02-design/service-contracts.md)
