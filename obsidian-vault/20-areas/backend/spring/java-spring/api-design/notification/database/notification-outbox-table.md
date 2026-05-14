---
title: "notification_outbox 테이블"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:50:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - database
  - outbox
---

# notification_outbox 테이블 ★

**[[database|↑ hub]]**

> Worker 가 스캔 / 발송 / retry 하는 큐 테이블. event_id UNIQUE + partial index.

---

## 1. Schema

```sql
CREATE TABLE notification_outbox (
    id                CHAR(26) PRIMARY KEY,
    event_id          VARCHAR(100) NOT NULL,            -- dedup 의 자연 키
    notification_type VARCHAR(50) NOT NULL,
    user_id           CHAR(26) NOT NULL,
    template_key      VARCHAR(100) NOT NULL,
    variables         JSONB NOT NULL DEFAULT '{}',
    channel_types     TEXT[] NOT NULL,                  -- ['FCM', 'APNS', 'EMAIL', 'IN_APP']
    priority          VARCHAR(20) NOT NULL DEFAULT 'NORMAL',
    aggregated        BOOLEAN NOT NULL DEFAULT FALSE,   -- "12명이 좋아요" 형
    aggregated_count  INTEGER,
    status            VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    attempts          INTEGER NOT NULL DEFAULT 0,
    max_attempts      INTEGER NOT NULL DEFAULT 5,
    last_error        TEXT,
    delivery_results  JSONB,                            -- per-channel 결과
    scheduled_at      TIMESTAMPTZ NOT NULL DEFAULT now(), -- quiet hours 지연
    next_attempt_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    sent_at           TIMESTAMPTZ,
    failed_at         TIMESTAMPTZ,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_outbox_status CHECK
        (status IN ('PENDING', 'PROCESSING', 'SENT', 'FAILED', 'DLQ')),
    CONSTRAINT chk_outbox_priority CHECK
        (priority IN ('LOW', 'NORMAL', 'HIGH', 'CRITICAL'))
);

CREATE UNIQUE INDEX ux_outbox_event_id ON notification_outbox (event_id);
CREATE INDEX ix_outbox_pending ON notification_outbox (next_attempt_at, priority DESC)
    WHERE status IN ('PENDING', 'PROCESSING');
CREATE INDEX ix_outbox_cleanup ON notification_outbox (sent_at)
    WHERE status = 'SENT' AND sent_at < now() - INTERVAL '30 days';
```

---

## 2. 컬럼 "왜"

### 2.1 `event_id` UNIQUE
- dedup ([[../design-decisions/dedup-strategy]])

### 2.2 `variables JSONB`
- template 변수 (`{userName, amount, ...}`)
- type 별 다른 변수 구조.

### 2.3 `channel_types TEXT[]`
- type 의 default + user preference 적용 후 실제 발송 채널 list.
- worker 가 router 로 전달.

### 2.4 `delivery_results JSONB`
- per-channel 결과 — push 성공, email 실패 시 분리 추적.

### 2.5 `scheduled_at` vs `next_attempt_at`
- `scheduled_at` = 예정 시간 (quiet hours 지연 등).
- `next_attempt_at` = 다음 시도 시각 (retry 포함).

### 2.6 `aggregated`
- batch aggregation 으로 만든 row 표시 ("N명이 좋아요").

### 2.7 partial index `pending`
- worker query: `WHERE status IN PENDING/PROCESSING AND next_attempt_at <= now()`.
- partial = SENT 수백만 row skip → fast.

---

## 3. Cleanup

```sql
-- 30일 지난 SENT row 삭제
DELETE FROM notification_outbox
WHERE status = 'SENT' AND sent_at < now() - INTERVAL '30 days';
```

→ in-app history 는 별도 `notifications` 테이블 ([[notifications-table]]).

---

## 4. 함정

### 함정 1 — event_id UNIQUE 없음
중복 발송.

### 함정 2 — variables 평문 (PII)
DB dump 시 PII 노출.
→ 민감 변수는 KMS encrypt (옵션).

### 함정 3 — SENT 영구 보관
무한 누적.
→ 30일 cleanup.

### 함정 4 — partial index 없음
worker query slow.

---

## 5. 관련

- [[database|↑ hub]]
- [[../design-decisions/outbox-pattern]]
- [[../implementation/outbox-worker-impl]]
- [[notification-dlq-table]]
