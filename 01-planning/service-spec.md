# 📋 모빌리티 정산 서비스 상세 기획서

> 이번 프로젝트의 **확정 기획서**다. 무엇을 왜 만드는지에 대한 기준 문서이며, 문제 4 "결제/정산의 멱등성과 정합성"에 대응한다.<br/>
> 프로젝트 채택 경위는 [archive/project-candidates.md](./archive/project-candidates.md) 참조.

---

## 1. 문제 재정의

모빌리티 플랫폼은 승차·이용이 끝날 때마다 승객에게 결제를 청구하고,<br/>
그 대금을 기사(또는 차량 소유자)에게 주기적으로 정산해 지급한다.<br/>
이 흐름에서 실무적으로 반복되는 두 가지 사고 유형이 있다.

1. **이중 결제**: 네트워크 타임아웃 재시도, 중복 클릭, 메시지 큐의 중복 전달 등으로 동일 결제 요청이 두 번 이상 처리된다.
2. **정산 불일치**: 기사에게 지급해야 할 금액과 플랫폼 원장(ledger)상의 금액이 어긋나, "누구에게 얼마를 줘야 하는가"를 신뢰할 수 없게 된다.

두 문제 모두 "돈이 걸린 시스템은 재시도해도 결과가 달라지면 안 된다(멱등성)"와 "장부의 합은 항상 맞아야 한다(정합성)"는 동일한 원칙으로 귀결되며,<br/>
이는 결제 서비스뿐 아니라 은행·전자지갑 등 핀테크 전반에서 쓰는 **이중기입 원장(double-entry ledger) 패턴**으로 해결한다.

- [Double-Entry Ledger Architecture for FinTech](https://medium.com/@gupta.rajneesh2010/20-double-entry-ledger-architecture-for-fintech-50d2ac6eb8e6)
- [Ledger System Design: Principles for Accuracy, Auditability, and Scale](https://fintechly.com/infrastructure/infrastructure-ledger-system-design/)

---

## 2. 시나리오

> 당신은 지역 기반 모빌리티 플랫폼 "D-Move"의 백엔드 개발팀에 합류했다.
>
> D-Move는 승객이 앱으로 차량을 호출하면 이용이 끝난 뒤 자동으로 카드 결제가 청구되고, 매일 자정 그날 운행한 기사들에게 운임의 80%를 정산해 지급하는 구조로 운영된다.
>
> 그런데 최근 두 가지 민원이 반복해서 들어왔다.
> - 승객 A는 결제 앱이 느려서 "결제하기"를 두 번 눌렀는데, 카드사 앱에는 **같은 운임이 두 번 청구**된 내역이 찍혔다.
> - 기사 B는 이번 주 정산 금액이 자신이 계산한 것과 달라 문의했는데, 운영팀도 원인을 바로 찾지 못했다. 정산 배치가 있었지만 **어떤 운행 건이 얼마로 계산됐는지 추적할 방법**이 마땅치 않았다.
>
> 팀 리드가 말했다.
> "이번 스프린트 목표는 명확해요.<br/>
> 첫째, 같은 결제 요청이 여러 번 와도 실제로는 딱 한 번만 청구되게 만들 것.<br/>
> 둘째, 정산 배치가 끝나면 '왜 이 금액이 나왔는지'를 누구나 원장만 보고 설명할 수 있게 만들 것."

---

## 3. 핵심 기능 명세

### 3.1 결제(Payment) 도메인

| 기능 | 설명 |
|---|---|
| 결제 요청 API | `POST /api/payments` — 운행(Trip) 종료 시 결제 청구.<br/>요청 헤더에 `Idempotency-Key`(클라이언트가 생성한 UUID)를 필수로 받는다. |
| 멱등 처리 | 동일한 `Idempotency-Key`로 재요청이 오면 실제 결제를 다시 실행하지 않고, **첫 번째 처리 결과를 그대로 반환**.<br/>DB의 `idempotency_key` 컬럼에 UNIQUE 제약을 걸어 동시 요청이 들어와도 하나만 성공하도록 강제. |
| 결제 상태 조회 | `GET /api/payments/{paymentId}` — 성공/실패/취소 상태 확인 |
| 결제 취소/환불 | `POST /api/payments/{paymentId}/cancel` — 취소 시 원장에 상쇄(역방향) 분개를 추가 기록(원본 삭제 금지) |

### 3.2 원장(Ledger) 도메인 — 이중기입 방식

| 기능 | 설명 |
|---|---|
| 분개 기록 | 결제 승인 시 하나의 거래(Transaction)에 대해 **차변(플랫폼 예수금 증가)**과<br/> **대변(기사 미지급금 증가)** 두 개의 분개(LedgerEntry)를 원자적으로 함께 기록 |
| 잔액 조회 | `GET /api/ledger/accounts/{accountId}/balance` — 계정(플랫폼/기사별)의 현재 잔액을 분개 합계로 계산 |
| 정합성 검증 | 특정 거래 또는 특정 기간의 모든 분개 합이 정확히 0인지 검증.<br/>→ 합이 0이 아니면 "정합성 이상" 알림 발생. |

### 3.3 정산(Settlement) 배치 — Spring Batch

| 기능 | 설명 |
|---|---|
| 일 단위 집계 Job | 매일 자정, 전날 완료된 결제 건을 기사별로 묶어 정산 금액(운임의 80%)을 계산 |
| Chunk 처리 | ItemReader(전날 결제 내역 조회) → ItemProcessor(기사별 집계·수수료 계산) → ItemWriter(정산 내역 저장)로 청크 단위 처리 |
| 정산 내역서 생성 | 기사별로 "어떤 운행 건이 얼마로 계산됐는지" 추적 가능한 명세(운행 ID 목록 + 금액)를 정산 항목(SettlementItem)에 기록 |
| 재실행 안전성 | 동일 배치 파라미터(정산 대상 일자)로 이미 성공한 Job은 재실행 시 예외를 발생시켜 중복 집계를 막고, 실패 시 실패 지점부터 재시작 |
| 지급 상태 관리 | 정산 확정(CONFIRMED) → 지급 완료(PAID) 상태 전이 관리 |

### 3.4 정산 대사(Reconciliation) 및 조회

| 기능 | 설명 |
|---|---|
| 대사 배치 | 정산 배치 종료 후, 원장상 기사 미지급금 합계와 정산 항목 합계가 일치하는지 자동 검증 |
| 관리자 조회 API | `GET /api/settlements?driverId=&date=` — 기사별/기간별 정산 내역 조회 |
| 불일치 알림 | 대사 결과 불일치 발견 시 로그/알림 (API 응답 코드나 관리자 대시보드 배지 시작, 확장 시 SNS 알림 연동) |

### 3.5 확장 기능 (선택, 시간이 남을 경우)

- 결제 수단 다변화(카드/포인트 혼합 결제) — 원장 계정 종류 확장으로 대응
- 정산 이의제기 프로세스(기사가 특정 건에 대해 재검토 요청)
- 정산 완료 이벤트를 SNS/SQS로 발행해 기사 앱에 푸시 알림 ([후보 F](./archive/candidate-f.md)와 결합 지점)

---


## 4. 아키텍처 — 현행은 별도 문서

이 기획서를 쓸 당시의 구성(단일 서버 + Amazon RDS)은 **더 이상 현행이 아니다.**
지금은 결제·원장·정산 3개 서버가 분리돼 API로 통신하고, DB도 시스템마다 하나씩 둔다.

| 알고 싶은 것 | 현행 문서 |
|---|---|
| 전체 구성, 서버·DB 분리 | [architecture.md §1](../02-design/architecture.md#1-전체-구성) |
| 서버 간 호출 계약, 공통 규약 | [service-contracts.md](../02-design/service-contracts.md) |
| 테이블 구조 | [erd.md](../02-design/erd.md) |
| 기술 선택과 인프라 | [architecture.md §5](../02-design/architecture.md#5-인프라) |
| 트랜잭션 경계, 결제-원장 원자성 | [architecture.md §3](../02-design/architecture.md#3-결제와-원장의-원자성) |

기획 단계에서 세운 원칙 중 **지금도 그대로 유효한 것**은 다음 둘이다.

- **멱등성** — `idempotency_key` UNIQUE 제약으로 동시 중복 요청 중 하나만 성공시킨다. 외부 락 없이 DB 유일성 제약만으로 해결하는 것이 가장 단순하고 견고하다.
- **정합성** — 잔액을 직접 갱신하지 않고 **분개의 합으로 항상 재계산한다.** 그래야 잔액이 이상할 때 분개 이력을 replay해 원인을 추적할 수 있다.

---

## 5. 일정

주차별 일정과 통합 테스트 계획은 [schedule.md](../05-milestones/schedule.md)에 있다.

---

## 6. 참고 자료

- [Double-Entry Ledger Architecture for FinTech](https://medium.com/@gupta.rajneesh2010/20-double-entry-ledger-architecture-for-fintech-50d2ac6eb8e6)
- [Ledger System Design: Principles for Accuracy, Auditability, and Scale](https://fintechly.com/infrastructure/infrastructure-ledger-system-design/)
- [How to Build a Bank Ledger in Golang with PostgreSQL using Double-Entry Accounting](https://www.freecodecamp.org/news/build-a-bank-ledger-in-go-with-postgresql-using-the-double-entry-accounting-principle/)
- [멱등성(Idempotency) 설계 가이드](https://velog.io/@2eunpal/%EB%A9%B1%EB%93%B1%EC%84%B1Idempotency-%EC%84%A4%EA%B3%84-%EA%B0%80%EC%9D%B4%EB%93%9C)
- [Spring Boot로 구축하는 회복력 있는(Resilient) 결제 시스템](https://velog.io/@anlee/Spring-Boot%EB%A1%9C-%EA%B5%AC%EC%B6%95%ED%95%98%EB%8A%94-%ED%9A%8C%EB%B3%B5%EB%A0%A5-%EC%9E%88%EB%8A%94Resilient-%EA%B2%B0%EC%A0%9C-%EC%8B%9C%EC%8A%A4%ED%85%9C-%EC%B9%B4%EB%93%9C-%EA%B2%B0%EC%A0%9C%EB%B6%80%ED%84%B0-%EC%A0%95%EA%B8%B0-%EA%B2%B0%EC%A0%9C-%EC%9E%A5%EC%95%A0-%EB%8C%80%EC%9D%91%EA%B9%8C%EC%A7%80)
- [Spring Batch 애플리케이션 성능 향상을 위한 주요 팁 - 카카오페이 기술 블로그](https://tech.kakaopay.com/post/spring-batch-performance/)
- [Spring Batch는 어떻게 Chunk 지향처리를 하고, Transaction을 언제 거는가](https://devocean.sk.com/blog/techBoardDetail.do?ID=164085)
