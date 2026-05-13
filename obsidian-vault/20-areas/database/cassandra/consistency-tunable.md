---
title: "Cassandra Tunable Consistency — CL / Quorum / Gossip / Repair"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:25:00+09:00
tags:
  - database
  - cassandra
  - consistency
---

# Cassandra Tunable Consistency

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | CL / Hinted Handoff / Read Repair / Anti-entropy |

**[[cassandra|↑ Cassandra hub]]**

---

## 1. 한 줄

Cassandra 는 **쓰기 / 읽기마다** consistency level (CL) 을 선택. 일관성 vs 가용성 의 트레이드를 매번 조정.

---

## 2. CL — 한눈에

| CL | 의미 |
| --- | --- |
| `ONE` | 1 노드만 |
| `TWO` / `THREE` | N 노드 |
| **`QUORUM`** | RF/2 + 1 노드 |
| **`LOCAL_QUORUM`** | 자기 DC 의 quorum (다중 DC 표준) |
| `EACH_QUORUM` | 모든 DC 의 quorum (강함) |
| `LOCAL_ONE` | 자기 DC 의 1 노드 |
| `ALL` | 모든 replica |
| `ANY` (write only) | hinted handoff 까지 OK |
| `SERIAL` / `LOCAL_SERIAL` | LWT |

### 2.1 R + W > N → Strong Consistency

```
RF = 3
W = QUORUM (2) + R = QUORUM (2) → 2 + 2 > 3 → 강한 일관성
W = ONE + R = ALL → 1 + 3 > 3 → 강함 (하지만 ALL 은 약함)
```

---

## 3. 다중 DC — LOCAL_QUORUM 권장

```
WAN latency 회피 + 강한 일관성
```

```python
session.execute(stmt, consistency_level=ConsistencyLevel.LOCAL_QUORUM)
```

EACH_QUORUM 은 다중 DC 모두 ack — 매우 느림 / DR 시 응답 불가.

---

## 4. Eventual Consistency

CL < ALL 이면 일부 replica 가 늦을 수 있음 → 다음 메커니즘으로 수렴:

1. **Hinted Handoff** — 죽은 노드 대신 다른 노드가 hint 저장 → 회복 시 적용
2. **Read Repair** — read 시 mismatch 발견하면 수정
3. **Anti-Entropy Repair** — 정기 `nodetool repair`

---

## 5. Hinted Handoff

```yaml
hinted_handoff_enabled: true
max_hint_window_in_ms: 10800000    # 3 시간
```

write 시 replica 노드가 down → 다른 노드가 hint 보관 → 회복 시 적용.

⚠️ down 시간 > max_hint_window → hint 폐기 → **repair 필요**.

---

## 6. Read Repair

```sql
ALTER TABLE users WITH read_repair_chance = 0.1;   -- 4.0 이전
ALTER TABLE users WITH read_repair = 'BLOCKING';   -- 4.0+
```

read 시 background / blocking 으로 mismatch 수정.

---

## 7. Anti-Entropy Repair

```bash
nodetool repair                       # 모든 keyspace
nodetool repair app                    # keyspace
nodetool repair app users              # table
nodetool repair --partitioner-range    # full repair 권장

# 운영
nodetool repair -pr                    # primary range 만 (분산)
```

**정기 (gc_grace_seconds 안에)** 필수. 안 하면 tombstone 살아남거나 데이터 inconsistency.

### 7.1 Reaper (운영 도구)
Cassandra Reaper — 자동 분산 repair. 운영 권장.

---

## 8. Tombstone — 삭제의 함정

```
DELETE FROM users WHERE user_id = ?;
→ tombstone (삭제 마커) 저장. 즉시 사라지지 X.
gc_grace_seconds (10일 기본) 후 compaction 으로 제거.
```

### 8.1 Tombstone 의 비용

```
SELECT 시 N tombstones 모두 읽어야 → 비효율
```

기본 `tombstone_warn_threshold = 1000`, `tombstone_failure_threshold = 100000`.

### 8.2 회피
- TTL 으로 자동 만료 (compaction 시 자연 제거 — TWCS 효율)
- 큰 row 단위 삭제 (partition / clustering range)
- batch 삭제 회피

---

## 9. Compaction

| Strategy | 의미 |
| --- | --- |
| **STCS** (SizeTieredCompactionStrategy) | 기본. 쓰기 위주 |
| **LCS** (LeveledCompactionStrategy) | 읽기 + 일정 latency |
| **TWCS** (TimeWindowCompactionStrategy) | 시계열 표준 |
| **UCS** (UnifiedCompactionStrategy, 5.0+) | 새 기본 후보 |

```sql
ALTER TABLE x WITH compaction = {
  'class':'LeveledCompactionStrategy',
  'sstable_size_in_mb': 160
};
```

자세히 → [[configuration#7-compaction-strategy]]

---

## 10. Snitch + Gossip

### 10.1 Gossip — 노드끼리 상태 교환

매 초 random 다른 노드에 자기 상태 push. → 곧 모든 노드가 클러스터 상태 알게 됨.

```bash
nodetool gossipinfo
```

### 10.2 Snitch — 토폴로지

자세히 → [[configuration#4-snitch--네트워크-토폴로지]]

---

## 11. LWT — Paxos

```sql
INSERT ... IF NOT EXISTS;
UPDATE ... IF col = old;
```

내부:
1. Prepare phase — 다른 LWT 와 충돌 없음 확인
2. Read phase — 현재 값 읽음
3. Propose phase — 새 값 제안
4. Commit phase — 적용

→ **4 round trip**. 같은 partition 안에서만.

`SERIAL` / `LOCAL_SERIAL` consistency 가 적용.

---

## 12. RF / CL 권장 표

| 시나리오 | RF | Write CL | Read CL |
| --- | --- | --- | --- |
| 단일 DC 운영 | 3 | QUORUM | QUORUM |
| 단일 DC 가벼움 | 3 | ONE | ONE |
| 다중 DC | per DC 3 | LOCAL_QUORUM | LOCAL_QUORUM |
| 강한 일관성 | per DC 3 | EACH_QUORUM | LOCAL_QUORUM |
| 로그 / 비핵심 | 2 | ONE | ONE |

---

## 13. CAP — Cassandra 의 위치

Cassandra = **AP** (Available + Partition tolerant).
하지만 CL 조정으로 **CP 처럼** 동작도 가능 (QUORUM + QUORUM).

→ "tunable" 의 의미.

---

## 14. 실패 / 복구 시나리오

### 14.1 1 노드 다운
- 나머지 노드가 가능한 CL 으로 응답
- Hinted Handoff 가 따라잡음
- 복구 시 자동 동기화

### 14.2 DC 전체 다운
- 다른 DC 가 LOCAL_QUORUM 으로 계속 동작
- 복구 시 repair 필요할 수도

### 14.3 Split Brain
Cassandra 는 AP — split 시 양쪽 다 write 받음. 통합 시 last-write-wins (timestamp 기준).

---

## 15. 모니터링

```bash
nodetool status              # 노드 UP/DN
nodetool ring                # token 분포
nodetool tablestats
nodetool tpstats
nodetool gossipinfo
nodetool netstats
nodetool compactionstats
nodetool getendpoints app users key
```

JMX / Prometheus jmx_exporter / DataDog.

---

## 16. 함정

### 16.1 CL=ALL 사용
한 노드 down → write 실패. 거의 사용 X.

### 16.2 EACH_QUORUM 운영
다중 DC 모두 ack — WAN latency. LOCAL_QUORUM 표준.

### 16.3 Repair 안 함
tombstone resurrection + inconsistency. 정기 repair (Reaper).

### 16.4 gc_grace_seconds = 0
tombstone 즉시 사라짐. repair 보장 못 하면 "delete 가 살아남음".

### 16.5 LWT 다용
Paxos 비용 ↑. 다른 모델로.

### 16.6 hinted handoff 의존
3 시간 넘어가면 의미 X. repair 가 본 대책.

### 16.7 read after write 기대 (ONE CL)
다른 replica 가 늦을 수도. QUORUM 사용.

### 16.8 timestamp 직접 지정
LWW 의존. timestamp drift / clock skew 위험.

---

## 17. 학습 자료

- **Cassandra Consistency** — datastax.com/blog
- **Reaper** — github.com/thelastpickle/cassandra-reaper
- **Cassandra: The Definitive Guide** Ch. 6

---

## 18. 관련

- [[replication-strategy]] — RF
- [[configuration]] — 운영
- [[cassandra]] — Cassandra hub
