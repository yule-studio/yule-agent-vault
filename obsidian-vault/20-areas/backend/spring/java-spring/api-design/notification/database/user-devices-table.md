---
title: "user_devices 테이블 — FCM/APNs/WebPush token"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:54:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - database
  - device
---

# user_devices 테이블 — FCM/APNs/WebPush token ★

**[[database|↑ hub]]**

> 자세한 schema + 컬럼 "왜" 는 [[../design-decisions/device-management#3]] 참고. 본 노트는 schema + index + cleanup query.

---

## 1. Schema

```sql
CREATE TABLE user_devices (
    id              CHAR(26) PRIMARY KEY,
    user_id         CHAR(26) NOT NULL,
    platform        VARCHAR(20) NOT NULL,        -- ANDROID / IOS / WEB / DESKTOP
    channel_type    VARCHAR(20) NOT NULL,        -- FCM / APNS / WEBPUSH
    token           TEXT NOT NULL,
    device_id       VARCHAR(100),                  -- FE 부여 UUID
    app_version     VARCHAR(50),
    os_version      VARCHAR(50),
    locale          VARCHAR(10) NOT NULL DEFAULT 'ko-KR',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'Asia/Seoul',
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    invalid_reason  VARCHAR(50),                   -- 'UNREGISTERED' / 'BAD_TOKEN'
    last_used_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_device_platform CHECK
        (platform IN ('ANDROID','IOS','WEB','DESKTOP')),
    CONSTRAINT chk_device_channel CHECK
        (channel_type IN ('FCM','APNS','WEBPUSH'))
);

CREATE UNIQUE INDEX ux_user_devices_token ON user_devices (token);
CREATE INDEX ix_user_devices_user ON user_devices (user_id, active);
CREATE INDEX ix_user_devices_cleanup ON user_devices (last_used_at) WHERE active;
```

---

## 2. UPSERT (token unique)

```sql
INSERT INTO user_devices (id, user_id, token, platform, ...)
VALUES (?, ?, ?, ?, ...)
ON CONFLICT (token) DO UPDATE
SET user_id = EXCLUDED.user_id,
    last_used_at = now(),
    updated_at = now(),
    active = TRUE,
    invalid_reason = NULL;
```

---

## 3. Cleanup

```sql
-- 90일 미사용 deactivate
UPDATE user_devices SET active = FALSE
WHERE last_used_at < now() - INTERVAL '90 days' AND active = TRUE;

-- 1년 지난 inactive DELETE
DELETE FROM user_devices
WHERE active = FALSE AND updated_at < now() - INTERVAL '1 year';
```

---

## 4. 관련

- [[database|↑ hub]]
- [[../design-decisions/device-management]]
- [[../security/device-token-rotation]]
- [[../implementation/device-management-impl]]
