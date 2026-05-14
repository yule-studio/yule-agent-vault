---
title: "Encryption — at-rest + E2EE (옵션)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:22:00+09:00
tags: [backend, java-spring, api-design, chat, design-decisions, encryption]
---

# Encryption — at-rest + E2EE (옵션)

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault

| Mode | 적용 |
| --- | --- |
| **at-rest** ★ (기본) | TLS 전송 + DB column 평문 (admin 검색 가능) |
| **column encrypt** | 민감 room 의 content 만 pgcrypto |
| **E2EE** (옵션) | SECRET room 만 — Signal Protocol |

---

## 2. 왜 E2EE 옵션 (default X)

### 2.1 왜 default X

- E2EE → 서버는 메시지 내용 못 봄 → 검색 / 모더 / 신고 처리 X.
- 카톡 default 도 server-side (option 시 E2EE).
- 한국 사용자 90% 가 신고 / 모더 의존.

### 2.2 SECRET room E2EE

- Signal Protocol (X3DH + Double Ratchet).
- 디바이스 별 key pair — 서버는 public key 만 보관.
- 새 device 추가 시 기존 device 가 key share.

자세히: 본 vault F12+ 검토.

---

## 3. at-rest

- DB → AWS RDS encryption at rest (KMS).
- S3 → SSE-S3 또는 SSE-KMS.
- TLS 1.3 (WebSocket WSS).

---

## 4. 함정

1. **WS (not WSS)** → MITM.
2. **localStorage 에 메시지 raw** (FE) → XSS 시 누설.
3. **E2EE 검색 / 모더 미고려** → 사용자 신고 처리 X.
4. **server-side key rotation 안 함** → leak 시 모든 메시지 노출.

---

## 5. 관련

- [[design-decisions|↑ hub]]
- [[../security/encryption-at-rest]]
- [[room-types#34]] — SECRET
