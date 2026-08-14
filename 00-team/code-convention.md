# 📜 코드 컨벤션 설정

### 📃 예시

---

1. **네이밍**

	- **클래스** : **`PascalCase`** (예: `BankAccount`, `StudentGrade`)
	- **메서드·변수** : **`camelCase`** (예: `getBalance`, `getUserName`)
	- **상수** : **`UPPER_SNAKE_CASE`** (예: `MAX_SIZE`)
	- **패키지** : 전부 **소문자** (예: `com.hjtn.study`)

---

2. **포매터 설정**

	- IntelliJ 기본 포매터 사용 (**`Ctrl+Alt+L`** 습관화)
	- **들여쓰기** : 스페이스 4칸 (IntelliJ 기본값)
	- **한 줄 최대 길이** (옵션) : 120자 권장 (`Settings → Code Style → Java → Right margin` 설정)

---

3. **코드 스타일**

	- **중괄호** : Allman 스타일 (`{` 새 줄, `}` 새 줄)
	- **빈 줄** : 메서드 사이 1줄, 논리 블록 사이 1줄
	- **`import`** : 와일드카드(`import java.util.*`) 금지, 자동 정리 (`Ctrl+Alt+O`)

---

4. **주석**

	- **클래스·메서드** : Javadoc (`/** */`)
	- **로직 설명** : 인라인 주석 (`//`)
	- 당연한 코드엔 주석 달지 않기

---

5. **IntelliJ 단축키 습관화**

	- **`Ctrl+Alt+L`** : 코드 포맷 정리
	- **`Ctrl+Alt+O`** : import 정리

---

6. **IntelliJ 추천 설정**

	- `Settings → Inspections` : 경고 레벨 확인하며 코드 품질 습관화
	- EditorConfig (`.editorconfig`) 파일 하나 만들어 두면 설정 고정 가능
	- Save Actions 플러그인 : 저장 시 자동 포맷·import 정리
