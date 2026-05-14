---
title: "notification_templates 테이블"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:58:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - database
  - template
---

# notification_templates 테이블

**[[database|↑ hub]]**

---

## 1. Schema

```sql
CREATE TABLE notification_templates (
    id              CHAR(26) PRIMARY KEY,
    template_key    VARCHAR(100) NOT NULL,
    channel_type    VARCHAR(20) NOT NULL,
    locale          VARCHAR(10) NOT NULL,
    title           VARCHAR(200),
    body            TEXT NOT NULL,
    deeplink        VARCHAR(500),
    variables       TEXT[],
    version         INTEGER NOT NULL DEFAULT 1,
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    updated_by      CHAR(26),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (template_key, channel_type, locale, version)
);

CREATE INDEX ix_templates_active ON notification_templates (template_key, channel_type, locale)
    WHERE active = TRUE;
```

---

## 2. Lookup 우선순위

```
exact (template_key, channel, ko-KR) →
language (template_key, channel, ko) →
default (template_key, channel, en-US) →
throw
```

---

## 3. Audit

- 변경 시 `version` increment + 옛 row `active=false`.
- `updated_by` + `updated_at` 기록.

---

## 4. 관련

- [[database|↑ hub]]
- [[../design-decisions/template-i18n]]
- [[../implementation/template-impl]]
