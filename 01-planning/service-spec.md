# 📋 모빌리티 정산 서비스 상세 기획서

> 이번 프로젝트의 <u>**확정 기획서**</u>다. 무엇을 왜 만드는지에 대한 기준 문서이며, 문제 4 <u>"결제/정산의 멱등성과 정합성"</u>에 대응한다.<br/>
> 프로젝트 채택 경위는 [archive/project-candidates.md](./archive/project-candidates.md) 참조.

---

## 1. 문제 재정의

모빌리티 플랫폼은 승차·이용이 끝날 때마다 승객에게 결제를 청구하고,<br/>
그 대금을 기사(또는 차량 소유자)에게 주기적으로 정산해 지급한다.<br/>
이 흐름에서 실무적으로 반복되는 두 가지 사고 유형이 있다.

1. <u>**이중 결제**</u>: 네트워크 타임아웃 재시도, 중복 클릭, 메시지 큐의 중복 전달 등으로 동일 결제 요청이 두 번 이상 처리된다.
2. <u>**정산 불일치**</u>: 기사에게 지급해야 할 금액과 플랫폼 원장(ledger)상의 금액이 어긋나, "누구에게 얼마를 줘야 하는가"를 신뢰할 수 없게 된다.

두 문제 모두 "돈이 걸린 시스템은 재시도해도 결과가 달라지면 안 된다(멱등성)"와 "장부의 합은 항상 맞아야 한다(정합성)"는 동일한 원칙으로 귀결되며,<br/>
이는 결제 서비스뿐 아니라 은행·전자지갑 등 핀테크 전반에서 쓰는 <u>**이중기입 원장(double-entry ledger) 패턴**</u>으로 해결한다.

- [Double-Entry Ledger Architecture for FinTech](https://medium.com/@gupta.rajneesh2010/20-double-entry-ledger-architecture-for-fintech-50d2ac6eb8e6)
- [Ledger System Design: Principles for Accuracy, Auditability, and Scale](https://fintechly.com/infrastructure/infrastructure-ledger-system-design/)

---

## 2. 시나리오

> 당신은 지역 기반 모빌리티 플랫폼 "D-Move"의 백엔드 개발팀에 합류했다.
>
> D-Move는 승객이 앱으로 차량을 호출하면 이용이 끝난 뒤 자동으로 카드 결제가 청구되고, 매일 자정 그날 운행한 기사들에게 운임의 80%를 정산해 지급하는 구조로 운영된다.
>
> 그런데 최근 두 가지 민원이 반복해서 들어왔다.
> - 승객 A는 결제 앱이 느려서 "결제하기"를 두 번 눌렀는데, 카드사 앱에는 <u>**같은 운임이 두 번 청구**</u>된 내역이 찍혔다.
> - 기사 B는 이번 주 정산 금액이 자신이 계산한 것과 달라 문의했는데, 운영팀도 원인을 바로 찾지 못했다. 정산 배치가 있었지만 <u>**어떤 운행 건이 얼마로 계산됐는지 추적할 방법**</u>이 마땅치 않았다.
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
| 멱등 처리 | 동일한 `Idempotency-Key`로 재요청이 오면 실제 결제를 다시 실행하지 않고, <u>**첫 번째 처리 결과를 그대로 반환**</u>.<br/>DB의 `idempotency_key` 컬럼에 UNIQUE 제약을 걸어 동시 요청이 들어와도 하나만 성공하도록 강제. |
| 결제 상태 조회 | `GET /api/payments/{paymentId}` — 성공/실패/취소 상태 확인 |
| 결제 취소/환불 | `POST /api/payments/{paymentId}/cancel` — 취소 시 원장에 상쇄(역방향) 분개를 추가 기록(원본 삭제 금지) |

### 3.2 원장(Ledger) 도메인 — 이중기입 방식

| 기능 | 설명 |
|---|---|
| 분개 기록 | 결제 승인 시 하나의 거래(Transaction)에 대해 <u>**차변(플랫폼 예수금 증가)**</u>과<br/> <u>**대변(기사 미지급금 증가)**</u> 두 개의 분개(LedgerEntry)를 원자적으로 함께 기록 |
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

## 4. 아키텍처

### 4.1 시스템 구성도

> ⚠️ <u>**논의 필요!**</u> 아래는 단일 서버 + Amazon RDS 구성이다.<br/>
> 현재는 <u>**결제·원장·정산 3개 서버가 분리되어 서로 API로 호출**</u>하고, <u>**DB도 시스템마다 하나씩**</u> 두는 구조이다.<br/>
> → 현행 구성도: [02-design/architecture.md §1](../02-design/architecture.md#1-전체-구성)

```
[클라이언트: 승객 앱 / 관리자 콘솔]
        │  REST (Idempotency-Key 헤더)
        ▼
[Spring Boot 4 API 서버]
   ├─ PaymentController → PaymentService → LedgerService (동일 트랜잭션)
   ├─ SettlementController (조회용)
        │
        ▼
[Amazon RDS (MySQL/PostgreSQL)]
   - trips, payments, ledger_entries, settlement_batches, settlement_items
        ▲
        │  (배치 Job 실행: 매일 자정, EventBridge 스케줄로 트리거)
[Spring Batch 정산 Job (같은 애플리케이션 내 배치 모듈 또는 별도 배치 서버)]
        │
        ▼
[Amazon S3] ← 정산 내역서(CSV/PDF) 아카이빙
[Amazon CloudWatch] ← 배치 실행 로그/실패 알림 모니터링
```

### 4.2 데이터 모델 개요 (ERD 핵심 요약)

- **Trip**(운행) — id, riderId, driverId, fare(운임), completedAt
- **Payment**(결제) — id, tripId, idempotencyKey(UNIQUE), amount, status(SUCCESS/FAILED/CANCELED), requestedAt
- **LedgerAccount**(계정) — id, ownerType(PLATFORM/DRIVER), ownerId
- **LedgerEntry**(분개) — id, transactionId(같은 거래를 묶는 그룹 키), accountId, direction(DEBIT/CREDIT), amount, createdAt
- **SettlementBatch**(정산 배치 실행 이력) — id, targetDate, status(RUNNING/CONFIRMED/FAILED), executedAt
- **SettlementItem**(정산 상세) — id, settlementBatchId, driverId, tripIds(포함된 운행 목록), amount, payoutStatus(CONFIRMED/PAID)

> 핵심 설계 원칙: **"잔액을 직접 갱신하지 않고, 분개(LedgerEntry)의 합으로 항상 재계산"** → 이력을 replay해서 원인 추적 가능 ([Double-Entry Ledger Architecture for FinTech](https://medium.com/@gupta.rajneesh2010/20-double-entry-ledger-architecture-for-fintech-50d2ac6eb8e6))

### 4.3 기술 선택과 근거

> ⚠️ <u>**논의 필요!**</u> 아래 표의 <u>**정산 리포트 저장(S3)·배치 스케줄링(EventBridge)**</u> 미확정이다.<br/>
> 🚧 DB 제품(**AWS Aurora** 혹은 **Supabase**)은 아직 미정이다.<br/>
> → 현행 인프라: [02-design/architecture.md §5](../02-design/architecture.md#5-인프라)

| 요소 | 선택 | 근거 |
|---|---|---|
| 멱등성 보장 | `idempotency_key` UNIQUE 제약 + 애플리케이션 레벨 선(先)조회 | 외부 락 없이 DB 유일성 제약만으로 동시 중복 요청 중 하나만 성공시킬 수 있어 가장 단순하고 견고함 |
| 원장 정합성 | 이중기입(차변=대변) 구조 | 특정 계정 잔액이 틀렸을 때 "어느 거래에서 깨졌는지" 분개 단위로 추적 가능 |
| 정산 배치 | **Spring Batch** (Chunk 지향 처리) | 대량의 결제 건을 안전하게 나눠 처리하고, 실패 시 실패 지점부터 재시작하는 표준 패턴을 프레임워크가 제공.<br/>카카오페이 정산플랫폼팀도 Spring Batch 기반 정산 처리 성능 최적화 사례를 공개한 바 있음. |
| 동시 처리 성능 | Spring Boot 4 가상 스레드(`spring.threads.virtual.enabled=true`) | 결제 API는 DB I/O 대기가 대부분이라 가상 스레드로 별도 리액티브 전환 없이 동시 처리량을 늘릴 수 있음 |
| 정산 리포트 저장 | Amazon S3 | 정산 내역서를 감사(audit) 목적으로 장기 보관 |
| 배치 스케줄링 | Amazon EventBridge → 배치 Job 트리거 | 별도 스케줄러 서버 없이 관리형으로 매일 자정 실행 |
| 모니터링 | Amazon CloudWatch | 배치 실패·정합성 불일치를 로그 기반으로 감지 |

- 참고: [Spring Batch 애플리케이션 성능 향상을 위한 주요 팁 - 카카오페이 기술 블로그](https://tech.kakaopay.com/post/spring-batch-performance/)
- 참고: [Spring Batch는 어떻게 Chunk 지향처리를 하고 Transaction을 언제 거는가](https://devocean.sk.com/blog/techBoardDetail.do?ID=164085)

### 4.4 트랜잭션·동시성 전략

> 서버와 DB가 시스템마다 분리되어, <u>**하나의 트랜잭션은 한 시스템 안에서만 성립한다.**</u><br/>
> 경계를 넘는 원자성 처리 방식은 [02-design/architecture.md §3](../02-design/architecture.md#3-결제와-원장의-원자성)이 단일 출처이며, 🚧 <u>**08.15 회의에서 결정한다.**</u>

**트랜잭션 경계**

| 구간 | 트랜잭션 | 원자성 |
|---|---|---|
| 결제 서버 — `Payment` insert | 결제 DB 로컬 트랜잭션 | ✅ 보장 |
| 원장 서버 — `LedgerEntry` 차변/대변 2건 insert | 원장 DB 로컬 트랜잭션 | ✅ 보장 |
| 정산 서버 — `SettlementBatch`·`SettlementItem` 저장 | 정산 DB 로컬 트랜잭션 (Chunk 단위) | ✅ 보장 |
| <u>**결제 ↔ 원장**</u> — 결제 저장 + 분개 기록 | 없음 (HTTP 경계) | ❌ <u>**불가능**</u> |

1. **결제 생성**

	- `Payment` insert는 <u>**결제 DB의 로컬 트랜잭션**</u>으로 처리한다.
	- 원장 분개는 같은 트랜잭션에 넣을 수 없으므로 `POST /api/ledger/entries` 호출로 분리한다.
	- `idempotency_key` UNIQUE 제약 위반 시 재요청으로 간주하고 <u>**기존 결제 결과를 반환**</u>한다.
	- 🚧 원장 호출이 실패했을 때의 보상 방식(재시도 / 결제 롤백 / 사후 대사)은 미확정.

2. **원장 분개 기록**

	- 하나의 `transactionId`에 속한 차변·대변 분개를 <u>**원장 DB의 한 트랜잭션으로 함께 커밋**</u>한다. 절반만 기록되는 상태는 생기지 않는다.
	- 차변 합 ≠ 대변 합이면 커밋 전에 `400`으로 거부한다.

3. **정산 배치**

	- Spring Batch의 Chunk 트랜잭션 경계를 활용해 일정 건수(예: 100건)마다 커밋, 실패 시 해당 청크만 롤백 후 재시도한다.
	- Reader가 <u>**HTTP 호출**</u>이 되면서 결제 서버 장애가 곧 배치 실패로 이어진다. 🚧 재시도 정책 미확정.

4. **재실행 방지**

	- 동일 `targetDate` 파라미터로 이미 `CONFIRMED` 상태인 `SettlementBatch`가 있으면 Job 시작 전에 예외 처리한다.

**동시성 제어**

| 제약 | 위치 | 막는 것 |
|---|---|---|
| `payments.idempotency_key` UNIQUE | 결제 DB | 같은 키의 동시 결제 요청 중 하나만 성공 |
| `ledger_entries.transaction_id` UNIQUE | 원장 DB | 같은 거래로 두 번 호출돼도 분개 1세트만 |
| `settlement_batches.target_date` + `CONFIRMED` | 정산 DB | 같은 날짜 정산 배치의 중복 집계 |

> DB가 나뉘어 <u>**세 제약이 서로 다른 DB에 있다.**</u><br/>
> 하나가 걸려도 다른 DB의 작업은 롤백되지 않으므로, 각 서버가 <u>**자기 제약 위반을 스스로 처리**</u>해야 한다.

---

## 5. 개발 로드맵 (실제 개발 가능 기간 3~4주 기준)

> 프로그램 일정 기준: 6주차(07.27~07.31) 기획 마무리<br/>
> → **7주차(08.03~08.07) / 8주차(08.10~08.14) / 9주차(08.17~08.21)** 3주간 개발<br/>
> → 9주차 팀밋업(08.18) 중간 점검<br/>
> → 10주차 성과교류회(08.25) 최종 발표.

### 7주차 (08.03~08.07) — 결제 도메인 + 멱등성

- Day 1~2: `Trip`, `Payment`, `LedgerAccount`, `LedgerEntry` 엔티티 및 ERD 확정, RDS 스키마 마이그레이션
- Day 3~4: 결제 생성 API(`POST /api/payments`) 구현 — `Idempotency-Key` 처리, 결제+원장 분개를 한 트랜잭션으로 처리
- Day 5: 동시 중복 요청 테스트(같은 Idempotency-Key로 동시에 여러 요청 발사 → 결과가 1건만 생성되는지 검증), 단위/통합 테스트 작성
- **완료 기준(Definition of Done)**: 같은 Idempotency-Key로 10개 동시 요청을 보내도 실제 결제와 원장 분개는 정확히 1세트만 생성됨을 테스트로 증명

### 8주차 (08.10~08.14) — 원장 정합성 + 정산 배치 기본 구현

- Day 1~2: 계정 잔액 조회 API(분개 합계 계산), 결제 취소 시 상쇄 분개 처리
- Day 3~5: Spring Batch 정산 Job 구현 — Reader(전일 결제 조회)/Processor(기사별 집계)/Writer(`SettlementBatch`, `SettlementItem` 저장), 재실행 방지 로직 적용
- **완료 기준**: 특정 날짜에 대해 정산 배치를 두 번 실행하면 두 번째 실행은 명시적으로 거부되고, 정산 결과가 원장의 기사 미지급금 합계와 일치함을 확인

### 9주차 (08.17~08.21) — 대사·조회 API + 통합/발표 준비

- Day 1~2: 정산 대사(reconciliation) 검증 로직 + 관리자 조회 API(`GET /api/settlements`)
- Day 3: EventBridge로 배치 스케줄 자동화, S3 정산 리포트 업로드
- Day 4: 통합 테스트, AWS 배포(RDS/EC2 또는 Elastic Beanstalk), 데모 시나리오(승객 결제 → 정산 배치 실행 → 대사 결과 확인) 리허설
- Day 5(팀밋업 08.18 전후): 발표자료 준비, 아키텍처 다이어그램 정리
- **완료 기준**: "이중 결제 방지"와 "정산 정합성 검증"을 라이브 데모로 직접 보여줄 수 있는 상태

---

## 6. 참고 자료

- [Double-Entry Ledger Architecture for FinTech](https://medium.com/@gupta.rajneesh2010/20-double-entry-ledger-architecture-for-fintech-50d2ac6eb8e6)
- [Ledger System Design: Principles for Accuracy, Auditability, and Scale - Fintechly](https://fintechly.com/infrastructure/infrastructure-ledger-system-design/)
- [How to Build a Bank Ledger in Golang with PostgreSQL using Double-Entry Accounting](https://www.freecodecamp.org/news/build-a-bank-ledger-in-go-with-postgresql-using-the-double-entry-accounting-principle/)
- [Spring Boot로 구축하는 회복력 있는(Resilient) 결제 시스템](https://velog.io/@anlee/Spring-Boot%EB%A1%9C-%EA%B5%AC%EC%B6%95%ED%95%98%EB%8A%94-%ED%9A%8C%EB%B3%B5%EB%A0%A5-%EC%9E%88%EB%8A%94Resilient-%EA%B2%B0%EC%A0%9C-%EC%8B%9C%EC%8A%A4%ED%85%9C-%EC%B9%B4%EB%93%9C-%EA%B2%B0%EC%A0%9C%EB%B6%80%ED%84%B0-%EC%A0%95%EA%B8%B0-%EA%B2%B0%EC%A0%9C-%EC%9E%A5%EC%95%A0-%EB%8C%80%EC%9D%91%EA%B9%8C%EC%A7%80)
- [쿠팡, 아마존, 알리와 같은 기업들은 결제 시스템을 어떻게 만드는걸까? - Team JSON Delivery](https://team-json-delivery.github.io/posts/pay-system/)
- [멱등성(Idempotency) 설계 가이드](https://velog.io/@2eunpal/%EB%A9%B1%EB%93%B1%EC%84%B1Idempotency-%EC%84%A4%EA%B3%84-%EA%B0%80%EC%9D%B4%EB%93%9C)
- [Spring Batch 애플리케이션 성능 향상을 위한 주요 팁 - 카카오페이 기술 블로그](https://tech.kakaopay.com/post/spring-batch-performance/)
- [Spring Batch는 어떻게 Chunk 지향처리를 하고, Transaction을 언제 거는가](https://devocean.sk.com/blog/techBoardDetail.do?ID=164085)
