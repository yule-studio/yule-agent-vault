---
title: "Encryption at rest"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:36:00+09:00
tags: [backend, java-spring, api-design, chat, security, encryption]
---

# Encryption at rest

**[[security|↑ hub]]**

---

- DB → AWS RDS encryption at rest (KMS).
- S3 (첨부) → SSE-S3 또는 SSE-KMS.
- TLS 1.3 (WSS + HTTPS).
- 옵션: SECRET room 의 column-level pgcrypto (content).

자세히: [[../design-decisions/encryption-strategy]].

---

## 관련

- [[security|↑ hub]]
- [[../design-decisions/encryption-strategy]]
