# 🎯 모빌리티 정산 서비스 프로젝트 개요

> 자세한 사항은 다음 참조.
> [모빌리티 정산 서비스 확정 기획서](./service-spec.md)

## 프로젝트명: **EasyADJ**

---

## 과정 개요

| 항목 | 내용 |
|---|---|
| 과정 | 2026 SW 파일럿 4기 — 모빌리티팀 |
| 대상 분야 | 모빌리티(Mobility) 백엔드 엔지니어링 |
| 확정 기술 스택 | Java 17+ / **Spring Boot 4**(Spring Framework 7) |
| 지원 자원 | AWS(Cloud Service) |
| 팀프로젝트 일정 | 6주차(07.27~07.31) 프로젝트 기획 + 기업설명회 → 7~9주차(08.03~08.21) 프로젝트 개발 → 9주차 팀밋업(08.18) → 10주차 성과교류회(08.25) 발표 |
| 실제 개발 가용 기간 | 약 3~4주 (7~9주차) |

- Spring Boot 4는 2025년 11월 정식 출시되었으며, Java 17을 최소 요구사항으로 유지하면서 `spring.threads.virtual.enabled=true` 한 줄로 가상 스레드(Virtual Threads)를 활성화할 수 있다.
- I/O 대기가 잦은 블로킹 코드(HTTP 호출, DB 조회, 외부 API 연동)를 리액티브 프로그래밍 없이도 대량 동시 처리할 수 있어, "대량의 동시 연결/요청을 다뤄야 하는" 모빌리티 백엔드 문제들과 특히 잘 맞는다.
  - [Baeldung - Spring Boot 4 & Spring Framework 7](https://www.baeldung.com/spring-boot-4-spring-framework-7)
  - [Virtual Threads in Spring Boot 4](https://medium.com/javarevisited/virtual-threads-in-spring-boot-4-what-actually-changes-for-your-code-9b490b57f400)

---

## 관련 문서

- 현행 아키텍처: [3서버 구성 + Supabase 공유 DB](../02-design/architecture.md)
- 개발 환경 설정: [dev-environment.md](../03-development/dev-environment.md)
- 후보 선정 배경: [모빌리티 백엔드 실무 문제 리스트](./archive/mobility-backend-problems.md) / [프로젝트 후보 선정](./archive/project-candidates.md)
- 팀 규칙: [코드 컨벤션](../00-team/code-convention.md) / [Git 컨벤션](../00-team/git-convention.md) / [GitHub 협업 방식](../00-team/github-workflow.md)
