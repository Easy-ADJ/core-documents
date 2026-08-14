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

<u>**3개 서버 레포에 동일한 라벨 세트 적용**</u>

| 라벨 | 용도 |
|---|---|
| `feat` | 기능 구현 |
| `bug` | 오류 수정 |
| `contract` | <u>**서버 간 API 계약**</u> 관련 (혼자 결정하면 안 됨) |
| `schema` | <u>**DB 스키마 변경**</u> |
| `blocked` | 다른 시스템 작업을 기다리는 중 |
| `question` | 결정이 필요한 논의 |

`contract`와 `schema`가 붙은 이슈는 <u>**[Service Contracts 문서](../02-design/service-contracts.md) 또는 [ERD 문서](../02-design/erd.md) 갱신까지 끝나야 close**</u>하기

---

## 4. 이슈 작성 시 포함할 것

별도 템플릿 파일을 두지 않는 대신 아래 내용으로 본문을 구성한다.

- <u>**무엇을**</u>: 한 줄 요약 (제목에 `[서비스명]`을 붙이면 통합 보드에서 알아보기 쉬움)
- <u>**왜**</u>: 어떤 상황에서 필요해졌는지
- <u>**누가 영향받는지**</u>: 다른 서버가 걸리면 담당자 멘션
- <u>**막혀 있다면**</u>: 무엇을 기다리는지 + 그 이슈 번호 (`blocked by Easy-ADJ/ledger-service#12`)
- <u>**계약·스키마 이슈는 변경 전/후를 나란히**</u> 적기

---

## 5. GitHub Projects 보드

<u>프로젝트 진행 현황은 **GitHub Projects 보드** 한 곳에서 관리한다.</u>

- 컬럼: `Backlog` / `Todo` / `In Progress` / `Blocked` / `Done`
- 이슈는 <u>**만들 때 바로**</u> 보드에 올리기
- 회의([04-meetings](../04-meetings/))는 이 보드를 함께 검토하는 것으로 시작

---

## 6. 브랜치와 PR

각 레포 공통으로 [Git 컨벤션 문서](./git-convention.md)의 브랜치 플로우(`main` ← `develop` ← `feat/*`)를 따른다.

- `main` 병합은 PR로만, 시스템 개발 담당자 판단 하에 혹은 중요한 기능이 포함된 경우 팀원 합의 하에 PR로 병합
- PR 본문에 관련 이슈 번호 적기 (`Closes #12`)
- <u>**시스템 간 계약을 바꾸는 PR은 상대 담당자를 리뷰어로 지정**</u><br/>
	계약 변경 절차는 [service-contracts.md §3](../02-design/service-contracts.md#3-계약-변경-절차) 참고.

---

## 관련 문서

- 커밋·브랜치 규칙: [git-convention.md](./git-convention.md)
- 팀 역할과 담당 서비스: [roles.md](./roles.md)
- 협업 도구 링크: [links.md](./links.md)
- 계약 변경 절차: [service-contracts.md](../02-design/service-contracts.md)
