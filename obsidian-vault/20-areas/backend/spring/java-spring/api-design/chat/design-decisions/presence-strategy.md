---
title: "Presence — online / away / offline (Redis TTL)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:12:00+09:00
tags: [backend, java-spring, api-design, chat, design-decisions, presence]
---

# Presence — online / away / offline (Redis TTL)

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault

| State | 조건 |
| --- | --- |
| ONLINE | WebSocket 연결 + 마지막 heartbeat < 1분 |
| AWAY | 연결 있지만 1~5분 inactive |
| OFFLINE | 연결 없음 (5분 timeout 후) |

Redis 만 (DB X — 너무 자주 변경).

---

## 2. Redis schema

```
presence:{userId} = ONLINE / AWAY / OFFLINE
TTL: 90s (heartbeat 마다 갱신)

presence:lastSeen:{userId} = 2026-05-14T15:00:00Z   (영구)
```

---

## 3. Heartbeat 흐름

```
client → SEND /app/heartbeat (every 30s)
  → Redis SET presence:{userId} ONLINE EX 90
  → if status changed → publish "presence:{userId}" + status

client typing → SEND /app/room/X/typing
  → presence ONLINE 갱신 (typing = active)
```

---

## 4. typing indicator

```
client → SEND /app/room/X/typing { typing: true }
server → /topic/room/X/typing { user: A, typing: true }
client (다른 user) → "A 가 입력 중..."

5초 후 자동 false (auto-stop).
```

---

## 5. 함정

1. **DB 에 presence 저장** → write 폭주.
2. **Redis 없으면 fail** → SPOF — Redis cluster.
3. **heartbeat 너무 자주** (10s) → 부하 ↑.
4. **마지막 접속 시각** 사용자 권한 → preference (보임 / 친구만 / 안 보임).
5. **typing broadcast** 매번 → debounce 1s (1번 보내고 5초 안 추가 없음).

---

## 6. 다른 컨텍스트

- 카톡: 마지막 접속 시각 (privacy 우선) — typing X.
- Slack: typing 표시 강.
- Discord: rich presence (게임 / 음악).

---

## 7. 관련

- [[design-decisions|↑ hub]]
- [[../implementation/presence-impl]]
- [[../enums/presence-status]]
