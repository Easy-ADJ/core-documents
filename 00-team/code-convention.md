# 📜 코드 컨벤션

## 네이밍

| 대상 | 규칙 | 예 |
|---|---|---|
| 클래스 | `PascalCase` | `BankAccount` |
| 메서드·변수 | `camelCase` | `getBalance` |
| 상수 | `UPPER_SNAKE_CASE` | `MAX_SIZE` |
| 패키지 | 전부 소문자 | `com.example.driverledgersystem` |

## 코드 스타일

- **중괄호**: Allman 스타일 (`{` 새 줄, `}` 새 줄)
- **들여쓰기**: 스페이스 4칸 (IntelliJ 기본값)
- **한 줄 길이**: 120자 권장 (`Settings → Code Style → Java → Right margin`)
- **빈 줄**: 메서드 사이 1줄, 논리 블록 사이 1줄
- **import**: 와일드카드(`import java.util.*`) 금지

## 주석

- 클래스·메서드는 Javadoc (`/** */`), 로직 설명은 인라인 (`//`)
- 당연한 코드에는 주석을 달지 않는다

## IntelliJ

- `Ctrl+Alt+L` 코드 포맷 정리, `Ctrl+Alt+O` import 정리 — 습관화한다
- Save Actions 플러그인을 쓰면 저장 시 자동으로 정리된다
- `.editorconfig`를 두면 설정이 고정된다
