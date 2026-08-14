# 🐙 GitHub 협업 방식

> 레포가 4개로 나뉜 상태에서 <u>**이슈를 어디에 내고, 현황을 어디서 보는지**</u>를 규정하고 있다.<br/>
> 자세한 <u>**커밋 메시지·브랜치 규칙**</u>은 [Git 컨벤션 문서](./git-convention.md) 참조.

---

## 1. 레포 구성

🏢 조직(Organization): [Easy-ADJ](https://github.com/Easy-ADJ)

| 레포 | 용도 | 담당 | 공개 |
|---|---|---|---|
| [core-documents](https://github.com/Easy-ADJ/core-documents) | 프로젝트 문서 전용 (이 레포) | 전원 | private |
| [driver-payment-system](https://github.com/Easy-ADJ/driver-payment-system) | payment 서비스 | 김주엽 | public |
| [ledger-system](https://github.com/Easy-ADJ/ledger-system) | ledger 서비스 | 이치헌 | public |
| [driver-settlement-system](https://github.com/Easy-ADJ/driver-settlement-system) | settlement 서비스 | 허진수 | public |

---

## 2. 이슈는 어디에?

- 기본 원칙: <u>**이슈는 각 시스템의 레포에 등록**</u> (문서 레포에는 문서 관련 이슈만)
- **시스템 간 문제**: <u>**고쳐야 하는 쪽(제공자) 레포에 내고, 그 서비스를 사용하는 관련자들 멘션하기**</u>

| 상황 | 어디에 | 멘션 |
|---|---|---|
| 원장 API 응답 형식을 바꿔야 함 | <u>**원장 레포**</u> | 그 API를 쓰는 결제·정산 담당자 |
| 정산 배치가 결제 API 호출에서 타임아웃 | <u>**결제 레포**</u> | 정산 시스템 담당자 |
| 내 서버 내부 버그 | <u>**내 레포**</u> | 없음 |
| DB 스키마 변경 | <u>**테이블 소유 시스템 레포**</u> | 선택 사항 |
| 프로젝트 구조 또는 문서 관련 문제 | <u>**core-documents**</u> | 관련자 혹은 나머지 2명 전원 |

---

## 3. 이슈 라벨

라벨은 <u>**이슈 템플릿이 자동으로 붙인다.**</u> (손으로 고르는 것은 `blocked`뿐)

### 서버 레포 3개 — 동일한 6개 세트

| 라벨 | 용도 |
|---|---|
| `feat` | 기능 구현 |
| `bug` | 오류 수정 |
| `contract` | <u>**서버 간 API 계약**</u> 관련 (혼자 결정하면 안 됨) |
| `schema` | <u>**DB 스키마 변경**</u> |
| `blocked` | 다른 시스템 작업을 기다리는 중 |
| `question` | 결정이 필요한 논의 |

### core-documents — 문서 레포 전용 5개 세트

서버 레포가 아니므로 `contract`·`schema` 이슈가 생기지 않는다.<br/>
대신 `question`·`blocked`는 <u>**이름을 그대로 공유**</u>해 통합 보드에서 한 번에 필터되게 한다.

| 라벨 | 용도 |
|---|---|
| `docs` | 문서 내용 추가·수정 |
| `fix` | 오탈자·깨진 링크·잘못된 내용 |
| `structure` | 문서 체계·폴더 구조 변경 |
| `question` | 결정이 필요한 논의 (회의 안건 후보) |
| `blocked` | 다른 결정·작업을 기다리는 중 |

> `blocked`는 새 이슈의 <u>**종류가 아니라 상태**</u>다. 그래서 템플릿이 없고, 기존 이슈에 손으로 붙인다.

`contract`와 `schema`가 붙은 이슈는 <u>**[Service Contracts 문서](../02-design/service-contracts.md) 또는 [ERD 문서](../02-design/erd.md) 갱신까지 끝나야 close**</u>하기

---

## 4. 이슈 템플릿

`New issue`를 누르면 템플릿 목록이 뜬다.

파일 위치: 각 레포의 `.github/ISSUE_TEMPLATE/`

| 레포 | 템플릿 | 필수 입력 |
|---|---|---|
| 서버 3개 | 🛠️ 기능 구현 | 무엇을 · 왜 |
| | 🐛 오류 수정 | 증상 · 재현 방법 · 기대 동작 |
| | 🔗 계약 변경 | 무엇을 · 왜 · <u>**변경 전/후**</u> · 영향받는 담당자 |
| | 🗄️ 스키마 변경 | 대상 테이블 · 왜 · <u>**변경 전/후 DDL**</u> · 영향받는 API |
| | ❓ 논의 | 논점 · 선택지와 트레이드오프 |
| core-documents | 📄 문서 추가·수정 | 대상 문서 · 무엇을 · 왜 |
| | 🔧 문서 오류 | 위치 · 잘못된 내용 · 올바른 내용 |
| | 🗂️ 구조 변경 | 무엇을 · 왜 · 영향 범위 |
| | ❓ 논의 | 논점 · 선택지와 트레이드오프 |

**설계 의도**

- 제목에 `[정산]` `[문서]` 같은 <u>**접두사가 자동으로 채워진다**</u> → 통합 보드에서 출처가 한눈에 보인다
- `contract`·`schema` 템플릿은 <u>**변경 전/후를 비우면 제출 자체가 막힌다**</u> → 말로만 설명해 상대가 다르게 이해하는 사고를 구조적으로 차단
- 논의 템플릿은 <u>**선택지와 트레이드오프가 필수**</u> → "어떻게 할까요?"만 있는 이슈는 회의에서 겉돈다

> 템플릿을 고쳐야 하면 해당 레포의 `.github/ISSUE_TEMPLATE/*.yml`을 수정한다.
> <u>**기본 브랜치(`main`)에 병합되어야 반영된다.**</u> 작업 브랜치에만 있으면 UI에 나타나지 않는다.

---

## 5. GitHub Projects 보드

<u>프로젝트 진행 현황은 **조직 레벨 GitHub Projects 보드** 한 곳에서 관리한다.</u>

- 컬럼(Status): `Backlog` / `Todo` / `In Progress` / `Blocked` / `Done`
- 회의([04-meetings](../04-meetings/))는 이 보드를 함께 검토하는 것으로 시작

### 자동화 설정 (완료)

이슈를 <u>**손으로 보드에 올릴 필요가 없다.**</u> 보드의 Workflows에 아래가 켜져 있다.

| 워크플로 | 동작 |
|---|---|
| Auto-add to project | 새 이슈를 보드에 자동 등록 |
| Item added to project | 상태를 `Todo`로 |
| Item closed | 상태를 `Done`으로 |

> ⚠️ <u>**레포를 보드에 link하는 것만으로는 이슈가 넘어오지 않는다.**</u>
> `Auto-add to project` 워크플로를 레포마다 켜야 한다. 새 레포가 생기면 보드 `⋯ → Workflows → Auto-add to project → Edit`에서 그 레포를 추가한다.

`blocked` 라벨을 붙였다면 보드의 Status도 `Blocked`로 함께 옮긴다. (라벨과 Status는 연동되지 않는다)

---

## 6. 브랜치와 PR

각 레포 공통으로 [Git 컨벤션 문서](./git-convention.md)의 브랜치 플로우(`main` ← `develop` ← `feat/*`)를 따른다.

- `main` 병합은 PR로만, 시스템 개발 담당자 판단 하에 혹은 중요한 기능이 포함된 경우 팀원 합의 하에 PR로 병합
- PR 본문에 관련 이슈 번호 적기 (`Closes #12`)
- <u>**시스템 간 계약을 바꾸는 PR은 상대 담당자를 리뷰어로 지정**</u><br/>
	계약 변경 절차는 [service-contracts.md §3](../02-design/service-contracts.md#3-계약-변경-절차) 참고.

### PR 템플릿

PR을 열면 본문이 자동으로 채워진다. 파일 위치: 각 레포의 `.github/pull_request_template.md`

| 레포 | 확인하는 것 |
|---|---|
| 서버 3개 | 변경 유형(커밋 prefix) · <u>**계약·스키마 영향**</u> · 비밀정보 포함 여부 · 남의 테이블 직접 접근 여부 |
| core-documents | 변경한 문서 · <u>**소유권**</u>(공동 소유 문서면 합의 근거) · 문서 작성 규칙 준수 |

체크박스를 지우지 말고 <u>**해당 없으면 "해당 없음"에 체크**</u>한다. 빈 항목과 확인 후 넘어간 항목은 리뷰어가 구분할 수 없다.

---

## 관련 문서

- 커밋·브랜치 규칙: [git-convention.md](./git-convention.md)
- 팀 역할과 담당 서비스: [roles.md](./roles.md)
- 협업 도구 링크: [links.md](./links.md)
- 계약 변경 절차: [service-contracts.md](../02-design/service-contracts.md)
