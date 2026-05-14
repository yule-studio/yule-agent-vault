---
title: "user_notification_preferences 테이블"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:56:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - database
  - preference
---

# user_notification_preferences 테이블

**[[database|↑ hub]]**

---

## 1. Schema

```sql
CREATE TABLE user_notification_preferences (
    user_id           CHAR(26) PRIMARY KEY,
    settings          JSONB NOT NULL DEFAULT '{}',
    quiet_hours_start TIME,
    quiet_hours_end   TIME,
    quiet_timezone    VARCHAR(50) NOT NULL DEFAULT 'Asia/Seoul',
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- JSONB query 가속 (옵션)
CREATE INDEX ix_pref_settings ON user_notification_preferences USING gin (settings);
```

---

## 2. settings JSONB 구조

```json
{
  "COMMENT_RECEIVED": { "FCM": true, "APNS": true, "EMAIL": false, "IN_APP": true },
  "LIKE_RECEIVED":    { "FCM": false, "APNS": false, "EMAIL": false, "IN_APP": true },
  "MARKETING":        { "FCM": false, "APNS": false, "EMAIL": false, "IN_APP": false },
  "PAYMENT_APPROVED": { "enforced": true },
  "SECURITY_ALERT":   { "enforced": true }
}
```

---

## 3. 기본값 (신규 사용자)

```java
public Map<String, Map<String, Boolean>> defaults() {
    return Map.of(
        "COMMENT_RECEIVED", Map.of("FCM", true, "EMAIL", false, "IN_APP", true),
        "LIKE_RECEIVED",    Map.of("FCM", false, "EMAIL", false, "IN_APP", true),
        "MARKETING",        Map.of("FCM", false, "EMAIL", false, "IN_APP", false)
    );
}
```

→ marketing default OFF — 신규 spam 방지.

---

## 4. 관련

- [[database|↑ hub]]
- [[../design-decisions/user-preferences]]
- [[../implementation/preference-impl]]
