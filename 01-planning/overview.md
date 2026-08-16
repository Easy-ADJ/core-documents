# 🎯 프로젝트 개요

**EasyADJ** — 모빌리티 정산 서비스.
자세한 내용은 [확정 기획서](./service-spec.md) 참조.

## 과정 정보

| 항목 | 내용 |
|---|---|
| 과정 | 2026 SW 파일럿 4기 — 자바 2팀 |
| 분야 | 모빌리티 백엔드 엔지니어링 |
| 기술 스택 | Java 17+ / Spring Boot 4 (Spring Framework 7) |
| 개발 기간 | 7~9주차 (08.03 ~ 08.21), 약 3주 |
| 팀밋업 | 9주차 08.18 |
| 최종 발표 | 10주차 08.25 (성과교류회) |

## 왜 Spring Boot 4인가

Spring Boot 4는 Java 17을 최소 요구사항으로 유지하면서 `spring.threads.virtual.enabled=true` 한 줄로 **가상 스레드**를 켤 수 있다.

I/O 대기가 잦은 블로킹 코드(HTTP 호출, DB 조회, 외부 API 연동)를 리액티브 전환 없이 대량 동시 처리할 수 있어, 동시 요청을 많이 다루는 모빌리티 백엔드에 잘 맞는다. **서버를 3개로 나눠 동기 REST 호출이 늘어난 지금 구조에서는 근거가 더 강해졌다.**

- [Baeldung — Spring Boot 4 & Spring Framework 7](https://www.baeldung.com/spring-boot-4-spring-framework-7)
- [Virtual Threads in Spring Boot 4](https://medium.com/javarevisited/virtual-threads-in-spring-boot-4-what-actually-changes-for-your-code-9b490b57f400)

---

## 관련 문서

- 현행 아키텍처: [architecture.md](../02-design/architecture.md)
- 개발 환경: [dev-environment.md](../03-development/dev-environment.md)
- 팀 규칙: [코드 컨벤션](../00-team/code-convention.md) · [Git 컨벤션](../00-team/git-convention.md) · [GitHub 협업](../00-team/github-workflow.md)
- 후보 선정 배경: [archive/](./archive/) (현행 아님)
