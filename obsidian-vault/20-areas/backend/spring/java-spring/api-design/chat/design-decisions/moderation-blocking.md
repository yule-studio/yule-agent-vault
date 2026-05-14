---
title: "Moderation + Blocking — 신고 / 차단 / 자동 모더"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:24:00+09:00
tags: [backend, java-spring, api-design, chat, design-decisions, moderation, block]
---

# Moderation + Blocking — 신고 / 차단 / 자동 모더

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault

| 영역 | 정책 |
| --- | --- |
| 신고 (report) | 5회 신고 → 자동 모더 (admin 검토 대기) |
| 차단 (block) | 양방향 — A 가 B 차단 시 양쪽 모두 메시지 send/receive X |
| 자동 욕설 필터 | OPEN room 만 (옵션) |
| admin 강퇴 | room 별 admin / 본 vault 운영자 |
| 메시지 삭제 (admin) | hidden status, audit 5년 |

---

## 2. Block 흐름

```
A 가 B 차단:
  blocks INSERT (blocker=A, blocked=B)
  A 의 모든 room list 에서 B 와 의 DIRECT room → "차단됨" 표시
  A → 어떤 room 에서도 B 의 메시지 receive X (filter)
  B → A 의 메시지 send 시 server reject (403)
```

---

## 3. Block filter (필수 적용 위치)

| 위치 | 적용 |
| --- | --- |
| 메시지 send | sender 가 receiver 를 차단했는지? 또는 receiver 가 sender 를? |
| 메시지 receive (broadcast) | recipient 의 차단 list 에 sender 있으면 skip |
| room list | 차단 user 와의 DIRECT room 표시 X |
| presence | 차단 user 의 presence 안 받음 |
| 검색 | 차단 user 의 메시지 결과에서 제외 |

자세히: [[../security/blocked-user-filter]].

---

## 4. Schema

```sql
CREATE TABLE blocks (
    blocker_id  CHAR(26) NOT NULL,
    blocked_id  CHAR(26) NOT NULL,
    reason      VARCHAR(500),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (blocker_id, blocked_id),
    CHECK (blocker_id != blocked_id)
);

CREATE INDEX ix_blocks_blocked ON blocks (blocked_id);
```

---

## 5. 신고 모델

```
report 5회 / 같은 user → 자동 hidden (메시지 / room)
admin 검토 후 복원 또는 ban
사용자에게 자동 알림 (강제 ON — [[../../notification/design-decisions/channel-selection]])
```

---

## 6. 함정

1. **block filter 누락** (검색 / list / presence) → 위치마다 적용 필수.
2. **차단 후 다시 친구 추가 — block row 남음** → block 해제 endpoint.
3. **양방향 X (단방향)** → "내가 차단했지만 상대 메시지 봄" 사고.
4. **신고 의 자동 모더 임계값 너무 낮음** (1회) → 악용.
5. **admin 강퇴 audit X** → 5년 audit log.

---

## 7. 다른 컨텍스트

- 카톡: 차단 = 양방향.
- WhatsApp: 차단 = 양방향 + "마지막 접속" 보임 X.
- Slack: workspace 단위 admin 만 ban.

---

## 8. 관련

- [[design-decisions|↑ hub]]
- [[../security/blocked-user-filter]]
- [[../../board/security/moderation-impl|↗ board moderation]]
