---
title: "Message storage — RDB vs Mongo vs Cassandra"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:04:00+09:00
tags: [backend, java-spring, api-design, chat, design-decisions, storage]
---

# Message storage — RDB vs Mongo vs Cassandra

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault — PostgreSQL (RDB)

| 옵션 | 적용 | 본 vault |
| --- | --- | --- |
| PostgreSQL 16 + monthly partition | F0~F8 | ★ |
| MongoDB | F후 (수억 메시지) | X |
| Cassandra | 대형 platform (카톡 / 라인) | X |
| Kafka + projection | analytics | F10+ |

---

## 2. 왜 PostgreSQL

### 2.1 왜 필요

- 트랜잭션 일관성 (메시지 + 읽음 표시 + room.last_message_at 같은 TX).
- JOIN (room + members + last message) — chat 목록 빠름.
- 운영 단순 (기존 PostgreSQL 의 backup / replication).

### 2.2 언제 다른 DB 검토

| 트래픽 | DB |
| --- | --- |
| ~ 1000만 메시지 / day | PostgreSQL ★ |
| ~ 1억 메시지 / day | PostgreSQL + 적극 partition + read replica |
| ~ 10억 / day | Cassandra / ScyllaDB |
| 분석 / 검색 | Elasticsearch + Kafka projection |

---

## 3. Partition 전략

```sql
CREATE TABLE messages (
    id              CHAR(26) NOT NULL,
    room_id         CHAR(26) NOT NULL,
    sender_id       CHAR(26) NOT NULL,
    seq             BIGINT NOT NULL,
    type            VARCHAR(20) NOT NULL,
    content         TEXT,
    metadata        JSONB,
    reply_to_id     CHAR(26),
    status          VARCHAR(20) NOT NULL DEFAULT 'SENT',
    created_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- 월별 partition
CREATE TABLE messages_2026_05 PARTITION OF messages
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX ix_msg_room_seq ON messages_2026_05 (room_id, seq DESC);
CREATE INDEX ix_msg_room_created ON messages_2026_05 (room_id, created_at DESC);
```

### 3.1 왜 월별

- 평균 한국 user 메시지 (1k / day) × 100만 user = 10억 row / month.
- partition 별 query → index size 작음.
- 옛 partition → cold storage.

---

## 4. 함정

1. **partition 안 함** → 단일 테이블 100억 row → query / index 폭주.
2. **room JOIN 매번** → N+1 → batch fetch / cache room info.
3. **JSON 메시지 (전체 column 1개)** → 검색 어려움.
4. **soft delete** 안 함 → 사용자 본인 삭제 후 영구 복구 불가 (정책 결정).

---

## 5. 다른 컨텍스트

| platform | 선택 |
| --- | --- |
| 카톡 | 자체 NoSQL + Cassandra-like |
| 라인 | HBase 기반 |
| WhatsApp | Erlang + Mnesia → 다국적 |
| Slack | MySQL + Vitess + Solr |
| 일반 SaaS | PostgreSQL (본 vault) |

---

## 6. 관련

- [[design-decisions|↑ hub]]
- [[../database/messages-table]]
- [[retention-policy]]
