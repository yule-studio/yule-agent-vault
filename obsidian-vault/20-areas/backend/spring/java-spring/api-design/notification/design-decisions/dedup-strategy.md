---
title: "Dedup 전략 — event_id UNIQUE + 시간 window"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:33:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - design-decisions
  - dedup
---

# Dedup 전략 — event_id UNIQUE + 시간 window

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault — 2 단계 dedup

| 단계 | 방식 | 적용 |
| --- | --- | --- |
| **L1** | `event_id` UNIQUE (DB) | 같은 event 2번 INSERT 방지 |
| **L2** | `(user_id, type, dedup_key)` 1h window | "12명 좋아요" 형 burst 집계 |

---

## 2. 왜 / 안 하면 / 대안

### 2.1 왜 L1 (event_id UNIQUE)

**왜 필요**
- Listener 가 같은 event 2번 처리 (재시도 / race) — 두 번째 INSERT 실패.
- 별도 check 없이 DB 가 보장.

**안 하면**
- 사용자에게 같은 알림 2번 → "spam 같다" 인상.

### 2.2 왜 L2 (시간 window 집계)

**왜 필요**
- 인기 글 의 좋아요 분당 100 → 작성자에게 분당 100 알림 = spam.
- "12명이 좋아요" 1 개로 집계 = UX ↑.

**안 하면**
- 사용자가 알림 OFF → 다른 중요 알림 도 무시.

### 2.3 대안

| 모델 | 적용 |
| --- | --- |
| **L1 + L2** ★ (본 vault) | 일반 |
| L1 만 (집계 X) | 단순 SaaS |
| client dedup | unreliable |
| Redis bloom filter | scale ↑ |

---

## 3. L1 구현

```sql
ALTER TABLE notification_outbox ADD COLUMN event_id VARCHAR(100) NOT NULL;
CREATE UNIQUE INDEX ux_outbox_event_id ON notification_outbox (event_id);
```

```java
try {
    outbox.save(NotificationOutboxRow.of(eventId, ...));
} catch (DataIntegrityViolationException e) {
    log.info("dedup skip: {}", eventId);
    return;
}
```

### 3.1 event_id 생성 규칙

- 도메인 event 의 자연 키 + ULID:
  - `comment-created-${commentId}-${userId}`
  - `payment-approved-${paymentId}`
  - `chat-message-${messageId}`
- → 같은 도메인 event 가 2번 발행 시 같은 id.

---

## 4. L2 구현 (집계)

```sql
CREATE TABLE notification_dedup_window (
    dedup_key   VARCHAR(200) PRIMARY KEY,        -- user_id|type|target_id (예: "u123|LIKE|post456")
    count       INTEGER NOT NULL DEFAULT 0,
    first_at    TIMESTAMPTZ NOT NULL,
    last_at     TIMESTAMPTZ NOT NULL,
    expires_at  TIMESTAMPTZ NOT NULL
);

CREATE INDEX ix_dedup_expires ON notification_dedup_window (expires_at);
```

```java
public void enqueueAggregated(LikeReceived ev) {
    var key = "%s|LIKE|%s".formatted(ev.recipientId(), ev.postId());
    var window = repo.findOrCreate(key, Duration.ofHours(1));
    window.increment(now());

    if (window.count() == 1) {
        // 첫 좋아요 → 즉시 발송
        outbox.save(NotificationOutboxRow.of(...));
    } else if (window.count() == 10 || window.count() == 100) {
        // 10, 100 milestone — 집계 알림 발송
        outbox.save(NotificationOutboxRow.aggregated(
            ev.recipientId(), "LIKE_BURST", window.count()));
    }
    // else: 그 사이 — 발송 X (집계 대기)
}
```

→ 1시간 후 window expire → reset.

---

## 5. 함정

### 함정 1 — event_id UNIQUE 없음
같은 event 2번 → 중복 알림.

### 함정 2 — event_id 생성 random
재시도 시 다른 id → dedup X.
→ 자연 키 기반.

### 함정 3 — 집계 없음 (인기 글 spam)
사용자 알림 OFF → 다른 알림 무시.

### 함정 4 — dedup_key 충돌
다른 도메인 event 가 같은 key.
→ prefix (type 명) 명시.

---

## 6. 다른 컨텍스트

- 트위터: 좋아요 알림 dedup window 24h.
- Instagram: 모든 알림 burst 집계.
- Slack: thread reply 시 같은 thread 의 알림 dedup.

---

## 7. 관련

- [[design-decisions|↑ hub]]
- [[batch-aggregation]]
- [[outbox-pattern]]
- [[../database/notification-outbox-table]]
