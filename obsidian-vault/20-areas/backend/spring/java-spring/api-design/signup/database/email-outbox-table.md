---
title: "email_outbox 테이블 — outbox pattern + worker"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:10:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - database
  - outbox
  - email
---

# email_outbox 테이블 — outbox pattern + worker

**[[database|↑ database hub]]**

> 트랜잭션 안에서 INSERT, **별도 워커** 가 SMTP / SES 발송.
> 이 패턴을 안 쓰면 **메일 발송 실패가 비즈니스 트랜잭션을 끌어내리거나, 트랜잭션 안에서 SMTP 호출하다 DB 락 폭증** 한다.

---

## 1. Schema

```sql
-- V7__create_email_outbox.sql
CREATE TABLE email_outbox (
    id              CHAR(26) PRIMARY KEY,
    to_email        VARCHAR(254) NOT NULL,
    template        VARCHAR(50)  NOT NULL,             -- 'verification' / 'password-reset' / 'welcome'
    payload         JSONB        NOT NULL,             -- { link, code, ttl, ... }
    status          VARCHAR(20)  NOT NULL DEFAULT 'PENDING',
    attempts        INTEGER      NOT NULL DEFAULT 0,
    max_attempts    INTEGER      NOT NULL DEFAULT 5,
    last_error      TEXT,
    external_id     VARCHAR(100),                       -- SES messageId
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    next_attempt_at TIMESTAMPTZ  NOT NULL DEFAULT now(),
    sent_at         TIMESTAMPTZ,
    failed_at       TIMESTAMPTZ,

    CONSTRAINT chk_email_outbox_status
        CHECK (status IN ('PENDING', 'PROCESSING', 'SENT', 'FAILED', 'DEAD_LETTER'))
);

-- 워커 polling 용 — partial index
CREATE INDEX ix_email_outbox_pending
    ON email_outbox (next_attempt_at, status)
    WHERE status IN ('PENDING', 'PROCESSING');

CREATE INDEX ix_email_outbox_created ON email_outbox (created_at DESC);
```

---

## 2. 각 컬럼의 "왜" — 4구조

### 2.1 `to_email VARCHAR(254) NOT NULL`

**왜 필요한가**
- 발송 대상 — 워커가 발송할 주소.
- RFC 5321 maximum 254.

**왜 user_id FK 가 아닌 별도 컬럼**
- 시점 의존성: outbox INSERT 시점의 user email 을 그대로 발송해야 함.
- 사용자가 발송 전에 email 변경하면 → 옛 email 로 발송해야 정상 (verification 의 의도된 흐름).
- user 가 hard delete 되어도 옛 outbox 는 발송 가능 (있는 큐 처리).

**안 하면 무슨 문제**
- user_id 만 저장 + 워커가 매번 join → user email 변경 시 새 email 로 잘못 발송.
- FK + ON DELETE CASCADE 시 → user 삭제 시 outbox row 도 삭제 → 마지막 인증 메일 사라짐.

**대안과 왜 안 됨**
- user_id + to_email 둘 다 → 중복 정보. 단 user 식별 audit 필요 시 유용. 본 vault 는 to_email 만.

**트레이드오프**
- to_email 만 → 어떤 user 의 outbox 인지 즉시 모름. application 의 payload 에 user_id 임베드 권장.

---

### 2.2 `template VARCHAR(50) NOT NULL`

**왜 필요한가**
- 워커가 발송 시 어떤 HTML/text 템플릿 사용할지 dispatch.
- `'verification'` / `'password-reset'` / `'welcome'` / `'announcement'`.

**왜 별도 컬럼 (payload 안에 type 안 둠)**
- 인덱싱 / 검색 / 메트릭 별도 가능 ("verification 메일 발송 성공률" 별도 집계).
- 워커의 router 분기 단순.

**안 하면 무슨 문제**
- 단일 큐에 type 없으면 → 워커가 payload 분석으로 type 추측 → maintainability ↓.
- 메트릭 / 알람을 type 별로 못 분리 → "verification 메일 발송 실패율 급증" 같은 알람 불가.

**대안과 왜 안 됨**
- payload.template 으로 → 가능하지만 JSONB 안 검색 부담. 별도 컬럼이 깔끔.

**트레이드오프**
- 50 char fixed → 충분. 향후 namespacing (`auth/verification`) 가능.

---

### 2.3 `payload JSONB NOT NULL`

**왜 필요한가**
- 각 template 별 동적 데이터 (`link`, `code`, `ttl`, `userName`, ...).
- 정적 구조로는 모든 template 의 변수 수용 불가.

**왜 JSONB (TEXT JSON 아님)**
- JSONB = 파싱 후 저장 → 검색 / 인덱싱 가능 (`WHERE payload->>'link' LIKE ...`).
- TEXT JSON = 매 조회 시 파싱.
- 디스크 크기는 JSONB 가 조금 더 큼 (parsed binary) 하지만 검색 성능 ↑.

**안 하면 무슨 문제 (TEXT 사용 시)**
- payload 검색 (디버깅 / audit) 시 매번 application 단 파싱.
- DB 의 GIN 인덱스 활용 불가.

**대안과 왜 안 됨**
| 후보 | 왜 안 됨 |
| --- | --- |
| 컬럼별 분리 (`link VARCHAR`, `code VARCHAR`, ...) | template 마다 사용 컬럼 다름 → NULL 다수 + ALTER TABLE 빈번 |
| JSON (PostgreSQL 의 text JSON) | 매 조회 파싱. JSONB 가 우위 |
| 별도 payload_data table | join 비용. 1:1 관계엔 같은 row 가 자연 |

**트레이드오프**
- JSONB schema-less → 잘못된 payload (link 누락 등) 가 통과 가능. application 단 검증 필수.
- 큰 payload (HTML 본문 임베드) 는 부담 — 링크 / 핵심 변수만 저장하고 본문은 worker 가 template 렌더링.

---

### 2.4 `status VARCHAR(20) NOT NULL DEFAULT 'PENDING'` + CHECK

**왜 5-state**

| state | 의미 | 전이 |
| --- | --- | --- |
| PENDING | 대기 (워커 미할당) | 초기 |
| PROCESSING | 워커가 처리 중 (race condition 방지 mark) | PENDING → |
| SENT | 발송 완료 | PROCESSING → |
| FAILED | 일시 실패 (재시도 예정) | PROCESSING → |
| DEAD_LETTER | 영구 실패 (max_attempts 초과) | FAILED → |

**왜 PROCESSING 필요**
- 다중 워커 환경 — 두 워커가 같은 row 동시 처리 시도 시 PROCESSING mark 로 race 방지.
- `SELECT FOR UPDATE SKIP LOCKED` 와 결합 — 다른 워커가 lock 잡은 row 는 skip.

**왜 FAILED 와 DEAD_LETTER 분리**
- FAILED = 재시도 예정 (next_attempt_at 활용).
- DEAD_LETTER = 포기 (운영자 수동 처리 필요).
- 메트릭 / 알람 분리.

**안 하면 무슨 문제**
- 단순 PENDING/SENT 만 → 재시도 메커니즘 / race condition 방어 X.
- DEAD_LETTER 없으면 → max_attempts 후에도 PENDING 으로 남음 → 무한 재시도 또는 silent ignore.

**대안과 왜 안 됨**
- 큐 미들웨어 (RabbitMQ / SQS) → 큐가 status 를 관리. 단 본 vault 는 DB 기반 outbox (인프라 단순).

**트레이드오프**
- DB 큐 → throughput 한계 (메시지 큐 대비). 분당 수만+ 발송 시 SQS / RabbitMQ 로 이관.

---

### 2.5 `attempts INTEGER` + `max_attempts INTEGER DEFAULT 5`

**왜 필요한가**
- 재시도 카운터. max_attempts 초과 = DEAD_LETTER.
- 일시적 실패 (SES rate limit / 네트워크) 자동 복구.

**왜 max_attempts 5 (3 / 10 아님)**
- 5번 × exponential backoff (30s, 5m, 30m, 2h, 6h) = 약 9시간 동안 시도.
- 9시간 = 일시적 외부 장애 (SES outage 등) 충분 복구 시간.
- 3번 = 너무 짧음 (외부 장애 동안 모두 DEAD_LETTER).
- 10번 = 너무 김 (DEAD_LETTER 까지 며칠).

**왜 컬럼 별로 (config 아님)**
- 메일 종류별 다른 정책 가능 (verification = 5번, announcement = 3번).
- 운영자가 row 단위 max_attempts 변경 가능 (특정 SaaS 의 일시 장애 대응).

**안 하면 무슨 문제**
- max_attempts 없으면 → 영원히 재시도 → DEAD_LETTER 없음 → 모니터링 시 "왜 안 됨" 분석 불가.
- attempts 카운터 없으면 → 같은 row 가 계속 시도 → SES API rate limit 도달.

**대안과 왜 안 됨**
- application config (예: max=5) → 모든 row 동일. 유연성 ↓.

**트레이드오프**
- exponential backoff 의 delay 정책 명확화 필요. 너무 짧으면 외부 장애 시 재폭주.

---

### 2.6 `last_error TEXT`

**왜 필요한가**
- debugging — 왜 발송 실패했는지 추적.
- DEAD_LETTER 시 운영자 수동 분석 자료.

**왜 TEXT (VARCHAR 아님)**
- SES error 메시지가 매우 김 (stack trace 포함 시 1000+ char).
- VARCHAR 길이 제한 → truncate 시 핵심 정보 손실.

**안 하면 무슨 문제**
- 발송 실패 후 "왜 실패했지?" 답 X. SES 콘솔 / log 만 의존.
- DEAD_LETTER 분석 / 재발송 결정 어려움.

**대안과 왜 안 됨**
- 별도 outbox_error_log → join 비용. 같은 row 가 자연.

**트레이드오프**
- TEXT 컬럼 → row 크기 ↑ → IO 부담. 단 SENT/DEAD_LETTER 후 cleanup 으로 정리.

---

### 2.7 `external_id VARCHAR(100)`

**왜 필요한가**
- SES / SendGrid 의 messageId 저장.
- bounce / complaint webhook 처리 시 매핑.

**언제 사용**
- SES SNS notification — bounce / complaint 받으면 messageId 로 outbox row 찾아서 audit.
- 사용자 CS — "메일 못 받았어요" 응답 시 messageId 로 SES 콘솔 추적.

**안 하면 무슨 문제**
- bounce webhook 받아도 어떤 row 인지 모름 → 모니터링 / 발송 reputation 관리 부실.
- 분쟁 시 "이 메일 발송됐다" 입증 시 SES 콘솔만 의존.

**대안과 왜 안 됨**
- 자체 ID 만 사용 → SES webhook 과 연계 X.

**트레이드오프**
- 100 char → SES messageId 충분. SendGrid / Mailgun 등 다른 provider 도 비슷.

---

### 2.8 `next_attempt_at TIMESTAMPTZ`

**왜 필요한가**
- 다음 시도 시점 — exponential backoff.
- 워커 polling: `WHERE next_attempt_at <= now() AND status IN ('PENDING','FAILED')`.

**왜 컬럼 (계산식 아님)**
- 매 polling 마다 attempts + delay 계산 부담.
- INSERT 시 한 번 계산 → 인덱스 lookup 만으로 polling 가능.

**안 하면 무슨 문제**
- next_attempt_at 없이 매 polling 마다 `WHERE created_at <= now() - attempts * delay` → 인덱스 활용 X → 풀스캔.
- 1000만 row 시 polling 자체가 수 초 → 발송 지연.

**대안과 왜 안 됨**
- 큐 미들웨어 사용 → 큐가 delay 관리. 인프라 부담.

**트레이드오프**
- attempts 증가 시 next_attempt_at 도 갱신 필요. application 책임.

---

### 2.9 `created_at` / `sent_at` / `failed_at`

**왜 모두 필요한가**
- created_at = 큐 enqueue 시점.
- sent_at = 발송 완료 시점 → 발송 지연 메트릭 (sent - created).
- failed_at = 영구 실패 시점 → DEAD_LETTER 분석.

**메트릭**
- `sent_at - created_at` = p99 발송 지연. 보통 1-5분.
- 지연 급증 = 워커 이상 또는 외부 API 문제.

**안 하면 무슨 문제**
- 발송 지연 메트릭 불가능.
- 분쟁 시 "정확히 언제 발송 시도/완료됐는지" 입증 X.

---

## 3. Outbox 패턴 — 왜 별도 table

```
[Application]
  @Transactional
  signup(...) {
      users.save(...);
      events.publish(UserRegistered);
  }
  ↓ COMMIT

[Listener — AFTER_COMMIT]
  on(UserRegistered) {
      outbox.enqueue(...);     ← 별도 트랜잭션 INSERT
  }

[Worker — 별도 프로세스]
  polling: outbox.findPending();
  for row: ses.send(row.toEmail, row.template, row.payload);
  if (success) row.status = SENT
  else row.attempts++, row.next_attempt_at = now + backoff
```

### 3.1 왜 outbox — 구체적 이유 4

**1. 메일 발송 실패가 비즈니스 트랜잭션 영향 X**
- 만약 같은 트랜잭션에서 SES 호출:
  - SES 가 5초 응답 → 트랜잭션 5초 락 점유 → DB connection pool 고갈.
  - SES 가 실패 → 비즈니스 트랜잭션 rollback → 회원가입 자체 실패.
- outbox 패턴:
  - 비즈니스 트랜잭션은 outbox INSERT 만 (수 ms).
  - 메일 발송은 워커가 비동기.

**2. 재시도 자동**
- 외부 SES 일시 장애 시 → 워커가 자동 재시도.
- 같은 트랜잭션 내 호출 시 → 실패 = 즉시 사용자 에러. 재시도 책임이 사용자 / client.

**3. DB 가 영속**
- 워커 / SMTP 서버 죽어도 큐 안 사라짐.
- 메시지 큐 (Kafka, RabbitMQ) 도 비슷하지만 outbox 는 추가 인프라 X.

**4. audit trail**
- "이 user 에게 발송된 모든 메일" SQL 한 줄로 조회.
- 메시지 큐 만 사용 시 별도 audit log 필요.

### 3.2 왜 AFTER_COMMIT (같은 트랜잭션 안 INSERT 아님)

**같은 트랜잭션 안 INSERT 시**
- outbox INSERT 실패 → 비즈니스 트랜잭션 rollback → 회원가입 실패.
- outbox 의 안정성이 비즈니스의 발목.

**AFTER_COMMIT 시 (본 vault)**
- 비즈니스 commit 후 별도 트랜잭션 INSERT.
- outbox INSERT 실패해도 비즈니스 영향 X.
- 단점: outbox INSERT 자체가 실패하면 → 메일 발송 안 됨 → application 단 retry 또는 fallback.

**언제 BEFORE_COMMIT 가 맞는가**
- 메일 발송이 비즈니스의 일부 (예: 결제 영수증 — 발송 안 되면 거래 무효) → BEFORE_COMMIT.
- auth 의 verification 메일은 안 가도 사용자가 재발송 요청 가능 → AFTER_COMMIT 안전.

자세히: [[../transactions]] § AFTER_COMMIT.

---

## 4. Worker 구현

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class EmailOutboxWorker {

    private final EmailOutboxRepository outbox;
    private final EmailClient client;                   // AWS SES / SendGrid
    private final Clock clock;

    @Scheduled(fixedDelay = 1000)                        // 1초마다
    @SchedulerLock(name = "emailOutboxWorker", lockAtMostFor = "5m")
    public void process() {
        var batch = outbox.findPending(50);              // 한 번에 50 row
        for (var row : batch) processOne(row);
    }

    @Transactional
    public void processOne(EmailOutboxRow row) {
        // 다른 워커 중복 처리 방지 — PROCESSING 으로 mark
        row.markProcessing(Instant.now(clock));
        outbox.save(row);

        try {
            var result = client.send(row.toEmail(), row.template(), row.payload());
            if (result.ok()) {
                row.markSent(result.externalMessageId(), Instant.now(clock));
            } else {
                row.recordFailure(result.errorMessage(), computeNextAttempt(row.attempts() + 1));
            }
        } catch (Exception e) {
            row.recordFailure(e.getMessage(), computeNextAttempt(row.attempts() + 1));
        }
        outbox.save(row);
    }

    private Instant computeNextAttempt(int attempt) {
        // Exponential backoff: 30s, 5m, 30m, 2h, 6h → DEAD_LETTER
        long delaySeconds = switch (attempt) {
            case 1 -> 30;
            case 2 -> 300;
            case 3 -> 1800;
            case 4 -> 7200;
            case 5 -> 21600;
            default -> -1;
        };
        return delaySeconds < 0 ? Instant.MAX : Instant.now(clock).plusSeconds(delaySeconds);
    }
}
```

### 4.1 왜 `@Scheduled(fixedDelay = 1000)` — 1초

- 너무 짧으면 (100ms) → DB 부담 + idle 시 무의미.
- 너무 길면 (1분) → 발송 지연.
- 1초 = 응답성 vs 부담 균형점.

### 4.2 왜 ShedLock 필수

- 다중 인스턴스 환경 (2+ 서버) — 같은 cron 시점에 워커 동시 실행.
- 같은 row 동시 처리 → SES 에 중복 발송 → spam reputation 손상.
- ShedLock 으로 단일 인스턴스만 실행 보장.

### 4.3 왜 batch 50

- 한 번에 더 많이 (1000) → 트랜잭션 길어짐 + 부분 실패 시 영향 큼.
- 적게 (10) → polling 자주 → 부담.
- 50 = 실용적 균형.

---

## 5. Cleanup

```sql
-- 30일 지난 SENT row 삭제
DELETE FROM email_outbox WHERE status = 'SENT' AND sent_at < now() - INTERVAL '30 days';

-- 30일 지난 DEAD_LETTER 도 — 단 audit 필요시 보존
DELETE FROM email_outbox WHERE status = 'DEAD_LETTER' AND failed_at < now() - INTERVAL '30 days';
```

**왜 30일**
- SENT — audit / debugging 기간 (CS 응대 "내가 작년에 받은 메일?" 같은 요청은 거의 없음).
- DEAD_LETTER — 운영자 수동 처리 / 통계 분석 기간.

**안 하면 무슨 문제**
- 월 수십만 row 누적 → 인덱스 비대 → polling 풀스캔 → 발송 지연.

---

## 6. JPA Entity

```java
@Entity
@Table(name = "email_outbox")
public class EmailOutboxJpaEntity {
    @Id @Column(length = 26) private String id;
    @Column(name = "to_email", nullable = false, length = 254) private String toEmail;
    @Column(nullable = false, length = 50) private String template;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(nullable = false, columnDefinition = "jsonb") private Map<String, Object> payload;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20) private EmailOutboxStatus status;

    @Column(nullable = false) private int attempts;
    @Column(name = "max_attempts", nullable = false) private int maxAttempts;
    @Column(name = "last_error", columnDefinition = "text") private String lastError;
    @Column(name = "external_id", length = 100) private String externalId;
    @Column(name = "created_at", nullable = false) private Instant createdAt;
    @Column(name = "next_attempt_at", nullable = false) private Instant nextAttemptAt;
    @Column(name = "sent_at") private Instant sentAt;
    @Column(name = "failed_at") private Instant failedAt;
}

public enum EmailOutboxStatus { PENDING, PROCESSING, SENT, FAILED, DEAD_LETTER }
```

> 💡 `@JdbcTypeCode(SqlTypes.JSON)` — Hibernate 6 의 JSONB 매핑. 옛 버전은 `JsonBinaryType` 등 별도 hibernate-types 의존.

---

## 7. 모니터링

| 메트릭 | 알람 기준 | 왜 |
| --- | --- | --- |
| `email_outbox.pending.gauge` | > 1000 | 워커 이상 / 큐 적체 |
| `email_outbox.sent.rate` | baseline 대비 50% ↓ | 발송 실패 급증 |
| `email_outbox.failed.rate` | > 10% | 외부 (SES) 문제 |
| `email_outbox.dead_letter.count` | 1시간 spike | 알람 + 수동 분석 필요 |
| `email_outbox.latency.p95` | > 5분 | 워커 throughput 부족 |

**왜 메트릭 분리**
- pending = 워커 throughput 문제 (워커 죽음 / DB 부담).
- failed = 외부 API 문제 (SES outage 등).
- dead_letter = 영구 실패 (분석 / 수동 처리 필요).

---

## 8. 함정 모음 — "이걸 안 하면 X 가 터짐"

### 함정 1 — 같은 트랜잭션 안에서 SMTP 호출
SES 응답 5초 → 트랜잭션 5초 락 → connection pool 고갈 → 회원가입 전체 멈춤.
→ outbox + AFTER_COMMIT 필수.

### 함정 2 — Worker 단일 실행 보장 안 함
다중 인스턴스에서 같은 row 동시 처리 → SES 중복 발송 → spam reputation 손상.
→ ShedLock 필수.

### 함정 3 — Retry 무한
DEAD_LETTER 없으면 실패 row 영원히 재시도 → SES rate limit + 메트릭 노이즈.
→ max_attempts 5 + DEAD_LETTER.

### 함정 4 — Cleanup 없음
SENT row 가 무한 누적 → polling 풀스캔 → 발송 지연.
→ daily cleanup.

### 함정 5 — `payload` 가 큰 JSON
HTML 본문 임베드 → row 크기 폭증 → IO 부담.
→ 핵심 데이터만 (link, ttl). 본문은 워커가 template 렌더링.

### 함정 6 — `external_id` 안 저장
SES bounce webhook 매핑 X → 모니터링 / 평판 관리 부실.
→ 발송 성공 시 messageId 저장.

### 함정 7 — `next_attempt_at` 인덱스 누락
polling 풀스캔 → 매 polling 마다 수 초.
→ partial index `ix_email_outbox_pending`.

### 함정 8 — Race condition (두 워커 같은 row)
PROCESSING mark + version (낙관 락) 또는 SELECT FOR UPDATE SKIP LOCKED.

```sql
SELECT * FROM email_outbox
WHERE next_attempt_at <= now() AND status = 'PENDING'
ORDER BY next_attempt_at
LIMIT 50
FOR UPDATE SKIP LOCKED;
```

→ PostgreSQL 의 `SKIP LOCKED` — 다른 워커가 lock 잡은 row 는 skip. 동시 처리 안전.

### 함정 9 — Backoff 정책 없음
즉시 retry → 외부 API 부담 + rate limit. Exponential backoff 필수.

### 함정 10 — Bounce / complaint 처리 안 함
무효 email 에 계속 발송 → IP / domain reputation 손상 → SES blacklist.
→ bounce webhook 처리 → 해당 user 의 email_verified_at 무효화 + 발송 stop.

### 함정 11 — Worker 가 batch 안 함
한 번에 1 row 처리 → polling 부담 ↑.
→ 50 row batch.

### 함정 12 — 트랜잭션 안에서 외부 API 호출하면 안 됨 (반복)
이게 outbox 패턴의 핵심 동기. 명시적으로 코드 리뷰 시 확인.

---

## 9. 관련

- [[database|↑ database hub]]
- [[../email-verification-impl]] — outbox 의 사용자
- [[../password-reset-impl]] — 같은 outbox
- [[../../webhook-send]] — 비슷한 outbox 패턴
- [[../../distributed-lock]] — ShedLock
- [[../transactions]] — AFTER_COMMIT 패턴
