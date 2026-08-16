# 🐙 GitHub 협업 방식

> 레포가 5개로 나뉜 상태에서 **이슈를 어디에 내고, 현황을 어디서 보는지**를 정한다.
> 커밋 메시지·브랜치 규칙은 [git-convention.md](./git-convention.md) 참고.

---

## 1. 레포 구성

조직: [Easy-ADJ](https://github.com/Easy-ADJ)

| 레포 | 용도 | 담당 | 공개 |
|---|---|---|---|
| [core-documents](https://github.com/Easy-ADJ/core-documents) | 프로젝트 문서 (이 레포) | 전원 | private |
| [driver-payment-system](https://github.com/Easy-ADJ/driver-payment-system) | payment 서비스 | 김주엽 | public |
| [driver-ledger-system](https://github.com/Easy-ADJ/driver-ledger-system) | ledger 서비스 | 이치헌 | public |
| [driver-settlement-system](https://github.com/Easy-ADJ/driver-settlement-system) | settlement 서비스 | 허진수 | public |
| [dashboard-system](https://github.com/Easy-ADJ/dashboard-system) | 기사·관리자 대시보드 | 허진수 | public |

---

## 2. 이슈는 어디에?

**이슈는 각 시스템의 레포에 등록한다.** 시스템 간 문제는 **고쳐야 하는 쪽(제공자) 레포에 내고** 그 API를 쓰는 사람을 멘션한다. 요청한 쪽 레포에 쌓으면 정작 고칠 사람이 안 본다.

| 상황 | 어디에 | 멘션 |
|---|---|---|
| 원장 API 응답 형식을 바꿔야 함 | 원장 레포 | 그 API를 쓰는 결제·정산 담당자 |
| 정산 배치가 결제 API에서 타임아웃 | 결제 레포 | 정산 담당자 |
| 내 서버 내부 버그 | 내 레포 | 없음 |
| DB 스키마 변경 | 테이블 소유 시스템 레포 | 영향받는 담당자 |
| 문서·프로젝트 구조 문제 | core-documents | 관련자 |

---

## 3. 이슈 라벨

라벨은 **이슈 템플릿이 자동으로 붙인다.** 손으로 고르는 것은 `blocked`뿐이다.

**서버 레포 3개**

| 라벨 | 용도 |
|---|---|
| `feat` | 기능 구현 |
| `bug` | 오류 수정 |
| `contract` | 서버 간 API 계약 (혼자 결정하면 안 됨) |
| `schema` | DB 스키마 변경 |
| `blocked` | 다른 시스템 작업을 기다리는 중 |
| `question` | 결정이 필요한 논의 |

**core-documents** — 서버 레포가 아니라 `contract`·`schema`가 없다. 대신 `docs`(내용 추가·수정) · `fix`(오탈자·깨진 링크) · `structure`(문서 체계 변경)를 쓰고, `question`·`blocked`는 이름을 그대로 공유해 통합 보드에서 한 번에 필터되게 한다.

- `blocked`는 이슈의 **종류가 아니라 상태**다. 그래서 템플릿이 없고 기존 이슈에 손으로 붙인다.
- `contract`·`schema` 이슈는 **[service-contracts.md](../02-design/service-contracts.md) 또는 [erd.md](../02-design/erd.md) 갱신까지 끝나야 close**한다.

---

## 4. 이슈 템플릿

`New issue`를 누르면 목록이 뜬다. 파일 위치는 각 레포의 `.github/ISSUE_TEMPLATE/`.

| 레포 | 템플릿 | 필수 입력 |
|---|---|---|
| 서버 3개 | 🛠️ 기능 구현 | 무엇을 · 왜 |
| | 🐛 오류 수정 | 증상 · 재현 방법 · 기대 동작 |
| | 🔗 계약 변경 | 무엇을 · 왜 · **변경 전/후** · 영향받는 담당자 |
| | 🗄️ 스키마 변경 | 대상 테이블 · 왜 · **변경 전/후 DDL** · 영향받는 API |
| | ❓ 논의 | 논점 · 선택지와 트레이드오프 |
| core-documents | 📄 문서 추가·수정 | 대상 문서 · 무엇을 · 왜 |
| | 🔧 문서 오류 | 위치 · 잘못된 내용 · 올바른 내용 |
| | 🗂️ 구조 변경 | 무엇을 · 왜 · 영향 범위 |
| | ❓ 논의 | 논점 · 선택지와 트레이드오프 |

의도한 효과가 셋 있다.

- 제목에 `[정산]` `[문서]` 같은 접두사가 자동으로 붙어 통합 보드에서 출처가 한눈에 보인다.
- `contract`·`schema` 템플릿은 **변경 전/후를 비우면 제출이 막힌다.** 말로만 설명해 상대가 다르게 이해하는 사고를 구조적으로 차단한다.
- 논의 템플릿은 선택지와 트레이드오프가 필수다. "어떻게 할까요?"만 있는 이슈는 회의에서 겉돈다.

> 템플릿을 고치려면 `.github/ISSUE_TEMPLATE/*.yml`을 수정하고 **`main`에 병합해야** 반영된다.

---

## 5. GitHub Projects 보드

진행 현황은 **조직 레벨 Projects 보드 한 곳**에서만 본다.

- 컬럼: `Backlog` / `Todo` / `In Progress` / `Blocked` / `Done`
- 회의는 이 보드를 함께 검토하는 것으로 시작한다.
- 이슈를 손으로 올릴 필요가 없다. `Auto-add to project`(새 이슈 자동 등록) · `Item added to project`(→ Todo) · `Item closed`(→ Done)가 켜져 있다.

> ⚠️ **레포를 보드에 link하는 것만으로는 이슈가 넘어오지 않는다.**
> 새 레포가 생기면 보드 `⋯ → Workflows → Auto-add to project → Edit`에서 그 레포를 추가한다.

`blocked` 라벨을 붙였다면 보드 Status도 `Blocked`로 옮긴다. 라벨과 Status는 연동되지 않는다.

---

## 6. 브랜치와 PR

브랜치 플로우(`main` ← `develop` ← `feat/*`)는 [git-convention.md](./git-convention.md)를 따른다.

- `main` 병합은 PR로만 한다. 중요한 기능은 팀원 합의 하에 병합한다.
- PR 본문에 이슈 번호를 적는다. (`Closes #12`)
- **계약을 바꾸는 PR은 상대 담당자를 리뷰어로 지정한다.**

PR을 열면 `.github/pull_request_template.md`가 본문을 자동으로 채운다.

| 레포 | 확인하는 것 |
|---|---|
| 서버 3개 | 변경 유형 · **계약·스키마 영향** · 비밀정보 포함 여부 · 남의 테이블 접근 여부 |
| core-documents | 변경한 문서 · **소유권**(공동 소유면 합의 근거) · 문서 규칙 준수 |

체크박스를 지우지 말고 **해당 없으면 "해당 없음"에 체크**한다. 빈 항목과 확인 후 넘어간 항목을 리뷰어가 구분할 수 없다.

---

## 관련 문서

- 커밋·브랜치 규칙: [git-convention.md](./git-convention.md)
- 팀 역할: [roles.md](./roles.md)
- 계약 변경 절차: [service-contracts.md §3](../02-design/service-contracts.md#3-계약-변경-절차)
