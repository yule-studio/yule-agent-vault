---
title: "Retention — 메시지 / 첨부 보유"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:20:00+09:00
tags: [backend, java-spring, api-design, chat, design-decisions, retention]
---

# Retention — 메시지 / 첨부 보유

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault

| 데이터 | 보유 |
| --- | --- |
| 메시지 (TEXT) | 영구 (사용자 본인 삭제 가능) |
| 첨부 (S3) | 30일 → cold storage (S3 Glacier) |
| 5분 안 본인 삭제 | hard delete |
| 5분 후 본인 삭제 | soft delete (hidden) |
| 강퇴 / 차단 후 메시지 | 그대로 (audit) |
| Audit (admin action) | 5년 |

---

## 2. 왜 영구 (자체 cleanup X)

- 사용자 일생의 대화 영구 보관 = chat platform 의 신뢰.
- 본인 삭제만 가능.
- 카톡 / WhatsApp 도 영구 (별도 cleanup X).

---

## 3. 첨부 cold storage

```
30일 후:
  S3 Standard → S3 Glacier Flexible Retrieval
  접근 시 1~5분 retrieval
```

→ Cost ↓ + 옛 첨부 retrieval 가능.

---

## 4. 사용자 본인 삭제

| 시점 | 동작 |
| --- | --- |
| 5분 안 | hard delete (DB + S3) + broadcast |
| 5분 후 | soft delete (hidden) + broadcast |
| 본인 탈퇴 | 모든 메시지 본인 정보 anonymize |

---

## 5. 함정

1. **cleanup cron** 자체 → 데이터 손실 + 사용자 신뢰 ↓.
2. **soft delete 후에도 검색** 노출 → SELECT WHERE deleted_at IS NULL.
3. **첨부 영구 S3 Standard** → cost 폭주.
4. **본인 삭제 후 다른 사용자 미반영** → broadcast 필수.

---

## 6. 관련

- [[design-decisions|↑ hub]]
- [[../database/messages-table]]
- [[attachment-strategy]]
