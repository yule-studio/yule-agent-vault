---
title: "Cassandra Replication Strategy — Simple / NetworkTopology / Multi-DC"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:30:00+09:00
tags:
  - database
  - cassandra
  - replication
---

# Cassandra Replication Strategy

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | RF / NTS / Multi-DC |

**[[cassandra|↑ Cassandra hub]]**

---

## 1. Replication Factor (RF)

각 row 가 몇 번 복제되나. **Keyspace 단위**.

```sql
CREATE KEYSPACE app WITH REPLICATION = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 3, 'dc2': 3
};
```

운영 표준: **RF = 3** (per DC).

---

## 2. SimpleStrategy

```sql
CREATE KEYSPACE dev WITH REPLICATION = {
  'class': 'SimpleStrategy',
  'replication_factor': 3
};
```

- 단일 DC 만 (rack / DC 무시)
- **운영 X — 단일 DC dev/test 만**

---

## 3. NetworkTopologyStrategy (NTS) — 운영 표준

```sql
CREATE KEYSPACE app WITH REPLICATION = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 3, 'dc2': 3, 'dc3': 2
};
```

- DC 별 RF 지정
- rack-aware — 같은 row 의 replica 가 가능한 다른 rack 에

---

## 4. Token / Partitioner

```yaml
partitioner: org.apache.cassandra.dht.Murmur3Partitioner    # 기본 (5.0)
```

각 partition_key → MurmurHash → token (64-bit).
token ring 위에서 노드들이 token 범위 담당.

### 4.1 vnode (Virtual Node)

```yaml
num_tokens: 16
```

각 노드가 N 개의 token 범위 담당. (옛 256, 새로 16 권장).
→ 노드 추가 시 rebalance 균등.

---

## 5. Multi-DC 구성

### 5.1 cassandra-rackdc.properties (per node)

```
# DC1, rack1
dc=dc1
rack=rack1

# DC2, rack1
dc=dc2
rack=rack1
```

### 5.2 NetworkTopologyStrategy + LOCAL_QUORUM

```sql
CREATE KEYSPACE app WITH REPLICATION = {
  'class': 'NetworkTopologyStrategy',
  'us_east': 3,
  'eu_west': 3
};

-- 클라이언트
session.execute(stmt, consistency_level=LOCAL_QUORUM)
```

각 DC 안에서 quorum (2/3) → 강한 일관성 + 낮은 latency.

### 5.3 DC 별 워크로드 분리

- DC1 — OLTP 운영
- DC2 — 분석 (Spark)
- DC3 — DR

```sql
-- 분석 DC 에 더 적은 RF
{'us_east': 3, 'analytics': 2}
```

---

## 6. Cross-DC 트래픽

- Write — coordinator → 모든 DC 의 replica 에 전달
- Read — LOCAL_QUORUM 시 자기 DC 만 (WAN 회피)

```bash
# DC 간 throughput 제한 (운영)
nodetool setinterdcstreamthroughput 200
nodetool setstreamthroughput 200
```

---

## 7. Rack Awareness

같은 partition 의 replica → 가능한 다른 rack 에:

```
RF=3, racks=3
replica1 → rack1
replica2 → rack2
replica3 → rack3
```

rack 1 down → 다른 2 rack 의 replica 살아 있음.

---

## 8. 노드 추가 / 제거

### 8.1 추가

```yaml
# new node
seeds: "existing-seed1,existing-seed2"
auto_bootstrap: true
```

```bash
systemctl start cassandra
nodetool status              # UP, NORMAL (조인 후)
nodetool cleanup             # 다른 노드의 남은 데이터 정리
```

### 8.2 제거 (정상)

```bash
nodetool decommission        # 다른 노드로 데이터 이동
```

### 8.3 제거 (장애)

```bash
nodetool removenode <node-id>
nodetool assassinate <node-id>   # 강제 (위험)
```

---

## 9. 데이터 마이그레이션

### 9.1 nodetool rebuild

```bash
nodetool rebuild -- <source-dc>
```

새 DC 추가 시 데이터 채우기.

### 9.2 sstableloader

```bash
sstableloader -d node1,node2 /path/to/sstables/
```

다른 클러스터에서 SSTable 로드 (백업 복원).

---

## 10. CDC (Change Data Capture)

```yaml
cdc_enabled: true
cdc_raw_directory: /var/lib/cassandra/cdc_raw
```

```sql
CREATE TABLE ... WITH cdc = true;
```

→ 변경이 CDC log 에 기록. Debezium / Kafka 로 stream.

---

## 11. 백업

### 11.1 nodetool snapshot

```bash
nodetool snapshot -t backup-2026-05-13 app
# → /var/lib/cassandra/data/app/users-xxx/snapshots/backup-2026-05-13/
```

hardlink 이라 즉시 (디스크 X). 옮길 때 실제 복사.

### 11.2 SSTable backup

매 SSTable flush 후 보관:

```yaml
incremental_backups: true
```

`backups/` 폴더에 hardlink. 디스크 ↑ — 외부로 정기 이동.

### 11.3 매니지드
DataStax Astra / Instaclustr / AWS Keyspaces — 자동 백업.

---

## 12. Repair (필수)

자세히 → [[consistency-tunable#7-anti-entropy-repair]]

```bash
# 정기 — gc_grace_seconds 안에
# 운영은 Reaper 권장
```

---

## 13. 함정

### 13.1 SimpleStrategy 운영
NetworkTopology 사용.

### 13.2 SimpleSnitch 운영
GossipingPropertyFileSnitch.

### 13.3 RF=1
가용성 X. 단 1 노드 down → 데이터 손실.

### 13.4 RF > 노드 수
의미 없음. 노드 수 늘리거나 RF 조정.

### 13.5 num_tokens 256 (옛)
rebalance 비용. 16 권장.

### 13.6 새 DC 추가 후 rebuild 누락
DC 비어 있음.

### 13.7 같은 seed 모든 노드에
seed 는 부트스트랩 — 2-3 노드만 지정.

### 13.8 decommission 없이 노드 정지
다른 노드가 hint 쌓다가 max_hint_window 후 손실. 정상 decom.

---

## 14. 학습 자료

- **NetworkTopologyStrategy** — cassandra.apache.org/doc
- **Cassandra Architecture** — DataStax docs
- **The Last Pickle Blog** — 운영 노하우

---

## 15. 관련

- [[consistency-tunable]] — CL
- [[configuration]] — yaml / rack
- [[cassandra]] — Cassandra hub
