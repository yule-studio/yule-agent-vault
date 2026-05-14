---
title: "Room types — DIRECT / GROUP / OPEN / SECRET"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:02:00+09:00
tags: [backend, java-spring, api-design, chat, design-decisions, room]
---

# Room types — DIRECT / GROUP / OPEN / SECRET

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault — 4 type (단일 schema)

| Type | 설명 | max member | 검색 | E2EE |
| --- | --- | --- | --- | --- |
| **DIRECT** | 1:1 (두 user pair) | 2 | X | optional |
| **GROUP** | N 명 + 방장 (초대제) | 500 | X | X |
| **OPEN** | 누구나 검색 / 가입 | 1500 | O | X |
| **SECRET** | 비밀번호 / 초대링크 + E2EE | 100 | X | O |

---

## 2. 왜 단일 schema (분리 X)

- 비즈니스 로직 99% 동일 — 같은 message / read / presence.
- 차이는 검증 / 생성 흐름 만 (DIRECT 는 pair UNIQUE).
- → 단일 `rooms` 테이블 + `type` 컬럼.

---

## 3. 차이점

### 3.1 DIRECT (1:1)

```
UNIQUE (user1_id, user2_id) — 두 user 의 pair 1개만
member list: 2명 고정
방장 / admin 개념 X
초대 X (생성 시 두 user 지정)
```

### 3.2 GROUP

```
member: 방장 1 + admin N + member N (최대 500)
방장만 멤버 강퇴 / 방 삭제 가능
admin: 멤버 강퇴 + 모더
초대제 (방장 / admin 만 추가 가능)
방 이름 / 프로필 가능
```

### 3.3 OPEN (오픈채팅)

```
검색 가능 (이름 / 태그)
누구나 가입 (비밀번호 옵션)
member 최대 1500 (카톡 표준)
모더 강 (욕설 필터 + 자동 ban)
일정 시간 무활동 → kick (옵션)
```

### 3.4 SECRET

```
초대링크 만 (검색 X)
E2EE (option) — 자세히: [[encryption-strategy]]
member 100 max
```

---

## 4. Schema

```sql
CREATE TABLE rooms (
    id              CHAR(26) PRIMARY KEY,
    type            VARCHAR(20) NOT NULL,
    name            VARCHAR(200),
    description     TEXT,
    profile_image_url VARCHAR(500),
    max_members     INTEGER NOT NULL,
    creator_id      CHAR(26),
    is_searchable   BOOLEAN NOT NULL DEFAULT FALSE,    -- OPEN
    password_hash   VARCHAR(200),                       -- SECRET / OPEN protected
    e2ee_enabled    BOOLEAN NOT NULL DEFAULT FALSE,
    last_message_at TIMESTAMPTZ,
    version         BIGINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ,

    CONSTRAINT chk_room_type CHECK (type IN ('DIRECT','GROUP','OPEN','SECRET'))
);

-- DIRECT room 의 user pair UNIQUE (member_a < member_b 정렬)
CREATE UNIQUE INDEX ux_direct_pair ON room_members (room_id)
    WHERE (SELECT type FROM rooms WHERE id = room_id) = 'DIRECT';
-- 실제로는 별도 direct_rooms 테이블 권장 (UNIQUE pair)
```

자세히: [[../database/rooms-table]].

---

## 5. 함정

1. **DIRECT pair UNIQUE 없음** — 같은 두 user 의 room 2개.
2. **GROUP max_member 검증 없음** — 무한 초대.
3. **OPEN 검색 → DIRECT / SECRET 포함** — privacy 누설.
4. **방장 양도 안 됨** — 방장 탈퇴 시 방 영구 잠김.
5. **member 강퇴 권한 검증 누락** — 일반 member 가 강퇴.

---

## 6. 다른 컨텍스트

| 모델 | 적용 |
| --- | --- |
| 디스코드 server / channel | room 안에 sub-channel — 본 vault 와 다름 |
| Slack workspace / channel | 동일 |
| 라인 / 카톡 | 본 vault 4 type 동일 |
| 게임 길드 chat | GROUP + 권한 시스템 |

---

## 7. 관련

- [[design-decisions|↑ hub]]
- [[../database/rooms-table]]
- [[../enums/room-type]]
- [[../domain-model/room-aggregate]]
- [[encryption-strategy]] — SECRET
