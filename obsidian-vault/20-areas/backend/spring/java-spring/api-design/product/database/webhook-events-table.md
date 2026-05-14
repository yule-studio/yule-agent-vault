---
title: "webhook_events 테이블 — PG webhook idempotency"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - database
  - webhook
---

# webhook_events 테이블 — PG webhook idempotency

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[database|↑ hub]]**

> PG webhook 의 모든 수신 기록 — dedup 24h + audit.

---

## 1. Schema

```sql
-- V35__create_webhook_events.sql
CREATE TABLE webhook_events (
    id              CHAR(26) PRIMARY KEY,
    event_id        VARCHAR(100) NOT NULL,            -- PG 가 발급한 이벤트 ID
    provider        VARCHAR(20) NOT NULL,             -- TOSS / KCP / STRIPE / KAKAO
    event_type      VARCHAR(50) NOT NULL,             -- payment.approved / payment.canceled / ...
    signature       VARCHAR(200) NOT NULL,
    raw_body        TEXT NOT NULL,                    -- 원본 (audit + 재처리)
    processed       BOOLEAN NOT NULL DEFAULT FALSE,
    processed_at    TIMESTAMPTZ,
    error_message   TEXT,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX ux_webhook_provider_event ON webhook_events (provider, event_id);
CREATE INDEX ix_webhook_unprocessed ON webhook_events (received_at) WHERE processed = FALSE;
CREATE INDEX ix_webhook_cleanup ON webhook_events (received_at) WHERE received_at < now() - INTERVAL '30 days';
```

---

## 2. 컬럼 "왜"

### 2.1 UNIQUE (provider, event_id)

- dedup — 같은 이벤트 재수신 시 INSERT 실패 = 중복 처리 방어.

### 2.2 `raw_body TEXT`

- 추후 재처리 / 분쟁 시 evidence.
- TEXT (max 1MB).

### 2.3 `processed BOOLEAN`

- 수신 시 INSERT (processed=false) → handler 처리 후 UPDATE (processed=true).
- handler 실패 시 processed=false 로 남음 → 별도 cron 으로 재시도.

### 2.4 partial index `unprocessed`

- 미처리 webhook 빠른 조회.

---

## 3. 처리 흐름

```
1. POST /webhooks/toss
2. signature 검증
3. webhook_events INSERT (UNIQUE event_id) — 실패 시 중복
4. publishEvent
5. handler: payment 상태 동기화
6. webhook_events UPDATE processed=true
```

자세히: [[../design-decisions/webhook-strategy]] · [[../implementation/payment-webhook-impl]].

---

## 4. Cleanup

```sql
-- 30일 지난 webhook 삭제
DELETE FROM webhook_events WHERE received_at < now() - INTERVAL '30 days';
```

→ 분쟁 윈도우 (30일) 이후는 audit 외 불필요.

---

## 5. 함정

### 함정 1 — event_id UNIQUE 없음
중복 처리.
→ UNIQUE.

### 함정 2 — raw_body 안 저장
재처리 / 분쟁 시 evidence X.
→ raw_body.

### 함정 3 — processed BOOLEAN 만
중도 실패 시 어떤 단계인지 X.
→ processed_at + error_message.

### 함정 4 — cleanup 없음
1년 후 row 수백만 → query 느려짐.
→ 30일 cron.

---

## 6. 관련

- [[database|↑ hub]]
- [[payments-table]]
- [[../design-decisions/webhook-strategy]]
- [[../security/webhook-signature]]
- [[../implementation/payment-webhook-impl]]
