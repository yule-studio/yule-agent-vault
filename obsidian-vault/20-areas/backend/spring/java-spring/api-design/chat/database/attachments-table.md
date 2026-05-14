---
title: "attachments 테이블"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:40:00+09:00
tags: [backend, java-spring, api-design, chat, database, attachment]
---

# attachments 테이블

**[[database|↑ hub]]**

---

## Schema

```sql
CREATE TABLE attachments (
    id              CHAR(26) PRIMARY KEY,
    message_id      CHAR(26) NOT NULL,
    type            VARCHAR(20) NOT NULL,           -- IMAGE / VIDEO / AUDIO / FILE
    storage_url     VARCHAR(500) NOT NULL,           -- S3 / CloudFront
    thumbnail_url   VARCHAR(500),
    file_name       VARCHAR(200),
    file_size       BIGINT NOT NULL,
    mime_type       VARCHAR(100) NOT NULL,
    width           INTEGER,
    height          INTEGER,
    duration_ms     INTEGER,                          -- VIDEO/AUDIO
    waveform        JSONB,                            -- AUDIO
    storage_tier    VARCHAR(20) NOT NULL DEFAULT 'STANDARD',  -- STANDARD / GLACIER
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_att_message ON attachments (message_id);
CREATE INDEX ix_att_cold ON attachments (created_at) WHERE storage_tier = 'STANDARD';
```

---

## 30일 cold storage cron

```java
@Scheduled(cron = "0 0 4 * * *")
public void moveToGlacier() {
    var attachments = repo.findOlderThanInStandard(Duration.ofDays(30));
    for (var a : attachments) {
        s3.changeStorageClass(a.bucket(), a.key(), StorageClass.GLACIER);
        a.markGlacier();
        repo.save(a);
    }
}
```

---

## 관련

- [[database|↑ hub]]
- [[../design-decisions/attachment-strategy]]
- [[../implementation/attachment-impl]]
