---
title: "Outbox 패턴 — 트랜잭션 안 외부 호출 X"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:22:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - design-decisions
  - outbox
---

# Outbox 패턴 — 트랜잭션 안 외부 호출 X ★

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ hub]]**

> 도메인 트랜잭션 안에서 FCM / APNs / SES 직접 호출 = 절대 X.
> Outbox 행 INSERT + 별도 worker 가 send → DB commit 보장 + 재시도 + 멱등.

---

## 1. 본 vault 결정

```
@Transactional {
    // 도메인 변경
    repo.save(order);

    // event publish (DB INSERT 만)
    outbox.save(NotificationOutboxRow.of(...));
}
// commit
// ↓
// AFTER_COMMIT listener 가 worker 깨우거나 worker 가 5s 마다 scan

Worker (별도 tx):
    1. SELECT FOR UPDATE SKIP LOCKED
    2. 외부 호출 (트랜잭션 밖)
    3. 결과 UPDATE
```

---

## 2. 왜 / 안 하면 / 대안 / 트레이드오프

### 2.1 왜 outbox 패턴

**왜 필요**
- 도메인 commit 와 알림 발송 의 **at-least-once 일관성**.
- "주문은 됐는데 알림 안 옴" / "주문 rollback 됐는데 알림 갔음" 둘 다 방지.
- 외부 호출 (FCM/APNs/SES) 의 변동성 (timeout / rate limit / 일시 장애) 흡수.

**구현 원리**
1. 도메인 tx 안에서 outbox 행 INSERT (같은 DB).
2. tx commit → outbox 행 영속.
3. 별도 worker 가 outbox scan → 외부 호출 → 결과 UPDATE.
4. 실패 시 retry.

### 2.2 안 하면 어떤 문제

| 실수 | 사고 |
| --- | --- |
| 도메인 tx 안 FCM 호출 (동기) | DB 락 30s + cascade (PG / 사용자 timeout) |
| AFTER_COMMIT 시 동기 FCM | tx 는 OK but listener thread 가 외부 호출 — 재시작 시 손실 |
| @Async + in-memory queue | 서버 재시작 시 손실 |
| 직접 Kafka send (outbox 없이) | DB commit 후 Kafka timeout 시 event 손실 |
| 외부 호출 후 DB UPDATE | 외부 OK but DB 실패 시 발송 추적 X |

### 2.3 대안

| 패턴 | 일관성 | 비용 |
| --- | --- | --- |
| **Outbox + Worker** ★ (본 vault) | at-least-once | DB INSERT + scan cost |
| 동기 외부 호출 | exactly-once but 실패 시 도메인 rollback | DB 락 + UX ↓ |
| @Async (in-memory) | best-effort | 서버 재시작 손실 |
| Saga (compensating tx) | at-least-once + 분산 | 복잡 |
| Event sourcing | 강 | 큰 변경 |
| Outbox + CDC (Debezium) | at-least-once + 분산 | infra ↑ |

### 2.4 트레이드오프

- Outbox = 단순 + 신뢰 ↑ + scan latency (5s).
- 동기 호출 = 빠름 + 위험.
- 본 vault: **5s scan 의 약간의 latency** vs **exactly-once 시도** = 신뢰 우선.

---

## 3. Schema

```sql
CREATE TABLE notification_outbox (
    id              CHAR(26) PRIMARY KEY,
    event_id        VARCHAR(100) NOT NULL,
    notification_type VARCHAR(50) NOT NULL,
    user_id         CHAR(26) NOT NULL,
    template_key    VARCHAR(100) NOT NULL,
    variables       JSONB NOT NULL,             -- template 변수
    channel_types   TEXT[] NOT NULL,             -- ['FCM', 'APNS', 'EMAIL']
    priority        VARCHAR(20) NOT NULL DEFAULT 'NORMAL',
    status          VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    attempts        INTEGER NOT NULL DEFAULT 0,
    max_attempts    INTEGER NOT NULL DEFAULT 5,
    last_error      TEXT,
    next_attempt_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    sent_at         TIMESTAMPTZ,
    failed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_outbox_status CHECK
        (status IN ('PENDING', 'PROCESSING', 'SENT', 'FAILED', 'DLQ'))
);

CREATE UNIQUE INDEX ux_outbox_event_id ON notification_outbox (event_id);
CREATE INDEX ix_outbox_pending ON notification_outbox (next_attempt_at, status)
    WHERE status IN ('PENDING', 'PROCESSING');
```

### 3.1 왜 event_id UNIQUE

- 같은 event 2번 listener 처리 시 (재시도 / race) 두 번째 INSERT 실패 = dedup.

### 3.2 왜 partial index (PENDING/PROCESSING)

- worker scan query: `WHERE next_attempt_at <= now() AND status IN ('PENDING', 'PROCESSING')`.
- partial = SENT / FAILED row 는 skip → 빠름.

자세히: [[../database/notification-outbox-table]].

---

## 4. Worker (SKIP LOCKED)

```java
@Component
@RequiredArgsConstructor
public class OutboxPublishWorker {

    private final NotificationOutboxRepository repo;
    private final ChannelRouter router;
    private final Clock clock;

    @Scheduled(fixedDelay = 1_000)    // 1s
    @SchedulerLock(name = "notifOutboxWorker", lockAtMostFor = "5m")
    public void publish() {
        var rows = pickup(100);     // batch
        for (var row : rows) processOne(row);
    }

    @Transactional
    public List<NotificationOutboxRow> pickup(int limit) {
        var rows = repo.findPendingForUpdateSkipLocked(limit, clock.now());
        rows.forEach(r -> r.markProcessing(clock.now()));
        repo.saveAll(rows);
        return rows;
    }

    public void processOne(NotificationOutboxRow row) {
        // 트랜잭션 밖
        try {
            var results = router.route(row);
            updateSuccess(row, results);
        } catch (TransientException e) {
            scheduleRetry(row, e);
        } catch (PermanentException e) {
            moveToDlq(row, e);
        }
    }

    @Transactional
    public void updateSuccess(NotificationOutboxRow row, List<DeliveryResult> results) {
        row.markSent(clock.now(), results);
        repo.save(row);
    }
}
```

자세히: [[../implementation/outbox-worker-impl]].

---

## 5. 함정

### 함정 1 — 직접 외부 호출 (outbox 없이)
DB commit 후 Kafka / FCM 호출 timeout → event 손실.
**Why**: DB tx 와 외부 호출 의 atomicity 불일치.
**How to apply**: outbox INSERT + worker.

### 함정 2 — Worker 안 외부 호출
DB 락 30s → 다른 worker 의 SELECT block.
**Why**: SELECT FOR UPDATE 의 lock 시간.
**How to apply**: pickup() tx 짧게 + 외부 호출 tx 밖.

### 함정 3 — event_id UNIQUE 없음
같은 event 2번 INSERT → 2번 발송.
**Why**: listener 의 race / retry 가능.
**How to apply**: UNIQUE 제약.

### 함정 4 — SENT row 영구 보관
1년 후 백만 row → query 느림.
**Why**: outbox 는 발송 보장 용 — 발송 후 audit 별도.
**How to apply**: 30일 cron cleanup + 영구는 audit 테이블.

### 함정 5 — SKIP LOCKED 없음
2 worker 가 같은 row pickup.
**Why**: SELECT 의 race.
**How to apply**: `FOR UPDATE SKIP LOCKED`.

### 함정 6 — Worker scan 너무 자주
DB CPU ↑ (불필요한 query).
**Why**: 1s vs 5s.
**How to apply**: 1s (실시간 알림) — but outbox lag metric 으로 조정.

---

## 6. 다른 컨텍스트

### 6.1 Kafka outbox (F8+ 분산)
- DB outbox + Debezium CDC → Kafka → 다중 consumer.
- 같은 패턴 but Kafka 가 중간 broker.
- 자세히: [[kafka-event-driven]].

### 6.2 Inbox 패턴 (반대)
- 외부 webhook 수신 시 inbox 테이블 dedup.
- 본 vault: [[../../product/database/webhook-events-table|↗ product webhook-events]].

### 6.3 Saga
- 다중 서비스 분산 트랜잭션 — 본 vault 의 단일 DB outbox 와 다름.

---

## 7. 관련

- [[design-decisions|↑ hub]]
- [[retry-strategy]]
- [[dedup-strategy]]
- [[../database/notification-outbox-table]]
- [[../implementation/outbox-worker-impl]]
- [[../transactions]]
- [[../../signup/database/email-outbox-table|↗ signup email-outbox (옛 단순 패턴)]]
