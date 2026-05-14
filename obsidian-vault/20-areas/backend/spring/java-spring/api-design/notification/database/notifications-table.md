---
title: "notifications 테이블 — in-app 영구"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:52:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - database
  - in-app
---

# notifications 테이블 — in-app 영구

**[[database|↑ hub]]**

> 사용자 "알림" 화면 의 영구 저장. outbox 와 별도 (outbox = 발송 보장, in-app = 사용자 history).

---

## 1. Schema

```sql
CREATE TABLE notifications (
    id              CHAR(26) PRIMARY KEY,
    user_id         CHAR(26) NOT NULL,
    type            VARCHAR(50) NOT NULL,
    title           VARCHAR(200) NOT NULL,
    body            TEXT NOT NULL,
    deeplink        VARCHAR(500),
    metadata        JSONB,
    image_url       VARCHAR(500),
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_notif_user_unread ON notifications (user_id, read_at, created_at DESC);
CREATE INDEX ix_notif_user_created ON notifications (user_id, created_at DESC);
```

---

## 2. 컬럼 "왜"

- `read_at` TIMESTAMPTZ — 언제 읽었는지 audit.
- `metadata JSONB` — type 별 동적 (postId / orderId / chatRoomId).
- `deeplink` — 클릭 시 화면 이동 URL.

---

## 3. API

```http
GET /api/v1/me/notifications?cursor=...&limit=20
PATCH /api/v1/me/notifications/{id}/read
PATCH /api/v1/me/notifications/read-all
DELETE /api/v1/me/notifications/{id}
```

---

## 4. 함정

- outbox 와 같은 테이블 → cleanup 시 사용자 history 삭제 → 별도.
- read_at BOOLEAN → "언제 읽음" X → TIMESTAMPTZ.
- unread count 매번 COUNT(*) → Redis cache 5분.
- cleanup 없음 → 90일 cron.

---

## 5. 관련

- [[database|↑ hub]]
- [[notification-outbox-table]]
- [[../implementation/in-app-channel-impl]]
