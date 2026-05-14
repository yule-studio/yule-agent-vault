---
title: "Read receipt — 카톡 '1/2/N' 읽음 표시 ★"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:08:00+09:00
tags: [backend, java-spring, api-design, chat, design-decisions, read-receipt]
---

# Read receipt — 카톡 '1/2/N' 읽음 표시 ★

**[[design-decisions|↑ hub]]**

> 카톡 표준: 메시지 옆 숫자 = "안 읽은 사람 수". 1:1 면 1/0, 그룹 9명 이면 9/8/7/.../0.

---

## 1. 본 vault — per-user per-message READ row

```sql
CREATE TABLE message_reads (
    message_id   CHAR(26) NOT NULL,
    user_id      CHAR(26) NOT NULL,
    read_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (message_id, user_id)
);

-- 또는: room 단위 last_read_seq (메모리 효율)
CREATE TABLE room_member_read_state (
    room_id           CHAR(26) NOT NULL,
    user_id           CHAR(26) NOT NULL,
    last_read_seq     BIGINT NOT NULL DEFAULT 0,
    last_read_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (room_id, user_id)
);
```

---

## 2. 왜 두 model (per-message vs last_read_seq)

### 2.1 per-message READ row

**장점**:
- 메시지 별 누가 봤는지 정확.
- 그룹 의 "본 사람 list" 표시 가능.

**단점**:
- 메시지 1개 × member 100명 = 100 row.
- 1000 메시지 × 100 = 10만 row / room → 누적.

### 2.2 last_read_seq (카톡 / 라인 추정)

**장점**:
- room 당 member 당 1 row — 매우 compact.
- "안 읽은 수 = total_messages - last_read_seq" 빠름.

**단점**:
- 메시지 별 본 사람 list 못 함 (카톡 그룹 의 "본 사람" 표시 X).

### 2.3 본 vault — **last_read_seq + (옵션) per-message** hybrid

- DIRECT / GROUP — last_read_seq 만 (default).
- 그룹 의 "본 사람 list 보기" 옵션 → per-message READ.

---

## 3. 카톡 의 "안 읽은 수" 계산

```
1:1:
  나(A) → B 의 last_read_seq < 내 message.seq → 1
                                              else 0

그룹 (10명):
  내 메시지 seq=100
  member 9명 중 last_read_seq >= 100 인 사람 수 = 7
  안 읽은 수 = 9 - 7 = 2
  → 카톡 표시 "2"
```

---

## 4. 읽음 처리

```
사용자 B 가 room 들어옴 / 메시지 보임:
  client: SEND /app/room/r1/read { last_seq: 100 }
  server: UPDATE room_member_read_state SET last_read_seq = 100
                                            WHERE room_id=r1 AND user_id=B
  server: publish "/topic/room/r1/read" { user_id: B, last_seq: 100 }
  client (다른 member): "안 읽은 수" 갱신
```

---

## 5. 멀티 디바이스 sync

- A 가 폰에서 읽음 → 태블릿에도 동기.
- last_read_seq 는 user 단위 (device 무관) → 자연스럽게 sync.

---

## 6. 함정

1. **per-message UNIQUE 없음** → 멀티 디바이스 동시 READ row 2개.
2. **last_read_seq 가 감소** (사용자가 위로 스크롤) → 표시 망가짐.
   → MAX(old, new) 만 update.
3. **읽음 broadcast 의 폭주** (대형 그룹) → 메시지 1 → 100 broadcast.
   → debounce 1s (1개 broadcast 로 합침).
4. **카톡 그룹 read count UI** 매번 query → cache 5초.
5. **읽음 OFF 사용자** → 그래도 last_read_seq 갱신 (서버 측 정확성).
   → broadcast 만 skip.

---

## 7. 다른 컨텍스트

- 카톡: last_read_seq (추정).
- WhatsApp: per-message ✓ (1/2/blue tick).
- Slack: thread / message level read marker.
- Discord: channel last read (mention 만 강조).

---

## 8. 관련

- [[design-decisions|↑ hub]]
- [[../database/message-reads-table]]
- [[../implementation/read-receipt-impl]]
- [[multi-device-sync]]
