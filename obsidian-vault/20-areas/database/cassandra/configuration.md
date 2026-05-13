---
title: "Cassandra 설정 — cassandra.yaml / JVM / 노드"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:10:00+09:00
tags:
  - database
  - cassandra
  - configuration
---

# Cassandra 설정

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 핵심 옵션 + JVM |

**[[cassandra|↑ Cassandra hub]]**

---

## 1. 설정 파일 위치

```
$CASSANDRA_HOME/conf/cassandra.yaml
$CASSANDRA_HOME/conf/jvm-server.options
$CASSANDRA_HOME/conf/jvm17-server.options    # JDK 17
$CASSANDRA_HOME/conf/cassandra-env.sh
$CASSANDRA_HOME/conf/cassandra-rackdc.properties
```

Linux 표준:
```
/etc/cassandra/
/var/lib/cassandra/
/var/log/cassandra/
```

---

## 2. cassandra.yaml — 핵심

```yaml
cluster_name: 'prod'
num_tokens: 16               # vnode 수 (옛 256, 최신 권장 16)
allocate_tokens_for_local_replication_factor: 3

# 디스크
data_file_directories:
  - /var/lib/cassandra/data
commitlog_directory: /var/lib/cassandra/commitlog
hints_directory: /var/lib/cassandra/hints
saved_caches_directory: /var/lib/cassandra/saved_caches

# 네트워크
listen_address: 10.0.0.1     # 클러스터 내부
rpc_address: 0.0.0.0          # 클라이언트
broadcast_address: 10.0.0.1
broadcast_rpc_address: 10.0.0.1
native_transport_port: 9042
storage_port: 7000

# Seed (클러스터 join)
seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
      - seeds: "10.0.0.1,10.0.0.2,10.0.0.3"

# Replication / Snitch
endpoint_snitch: GossipingPropertyFileSnitch

# 인증
authenticator: PasswordAuthenticator
authorizer: CassandraAuthorizer
role_manager: CassandraRoleManager

# Memtable / Cache
memtable_allocation_type: heap_buffers
memtable_heap_space_in_mb: 2048
key_cache_size_in_mb: 100
row_cache_size_in_mb: 0       # 보통 0 (사용 시 신중)

# Compaction
concurrent_compactors: 4
compaction_throughput_mb_per_sec: 64

# Read / Write
concurrent_reads: 32
concurrent_writes: 32
concurrent_counter_writes: 32

# Timeout
read_request_timeout_in_ms: 5000
write_request_timeout_in_ms: 2000
range_request_timeout_in_ms: 10000

# Hinted Handoff
hinted_handoff_enabled: true
max_hint_window_in_ms: 10800000   # 3 hours
```

---

## 3. JVM — jvm-server.options

```
# Heap
-Xms16G
-Xmx16G

# GC (G1 권장)
-XX:+UseG1GC
-XX:MaxGCPauseMillis=300
-XX:G1HeapRegionSize=16m

# 또는 ZGC (JDK 17+)
-XX:+UseZGC
```

### 3.1 Heap 크기

- **8-32 GB**
- 너무 크면 GC pause 길어짐
- ScyllaDB 는 JVM X (C++) → 더 큰 메모리 가능

### 3.2 Compressed OOPS
< 32 GB heap 권장 (Java).

---

## 4. Snitch — 네트워크 토폴로지

```yaml
endpoint_snitch: GossipingPropertyFileSnitch
```

| Snitch | 의미 |
| --- | --- |
| `SimpleSnitch` | 단일 DC dev |
| `GossipingPropertyFileSnitch` | 운영 표준 (DC / rack 명시) |
| `PropertyFileSnitch` | 옛 |
| `Ec2Snitch` / `Ec2MultiRegionSnitch` | AWS |
| `GoogleCloudSnitch` | GCP |

### 4.1 cassandra-rackdc.properties

```
dc=dc1
rack=rack1
```

각 노드에 다르게.

---

## 5. Replication Factor / Strategy

키스페이스 단위:

```sql
CREATE KEYSPACE app
WITH REPLICATION = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 3,
  'dc2': 3
};

-- 단일 DC dev (운영 X)
CREATE KEYSPACE dev
WITH REPLICATION = {
  'class': 'SimpleStrategy',
  'replication_factor': 3
};
```

자세히 → [[replication-strategy]]

---

## 6. Consistency 기본값

cqlsh / 드라이버에서:

```sql
CONSISTENCY QUORUM;
CONSISTENCY LOCAL_QUORUM;        -- 다중 DC 권장
```

자세히 → [[consistency-tunable]]

---

## 7. Compaction Strategy

```sql
CREATE TABLE ... WITH compaction = {
  'class': 'LeveledCompactionStrategy'
};
```

| Strategy | 의미 |
| --- | --- |
| `SizeTieredCompactionStrategy` (STCS, 기본) | 쓰기 위주 |
| `LeveledCompactionStrategy` (LCS) | 읽기 위주 + 일정 latency |
| `TimeWindowCompactionStrategy` (TWCS) | 시계열 |

### 7.1 TWCS — 시계열에 표준

```sql
WITH compaction = {
  'class': 'TimeWindowCompactionStrategy',
  'compaction_window_unit': 'DAYS',
  'compaction_window_size': 1
};
```

---

## 8. TTL / GC Grace

```sql
CREATE TABLE ... WITH
  default_time_to_live = 86400,           -- 24h
  gc_grace_seconds = 864000;              -- 10 days (default)
```

`gc_grace_seconds` = 삭제 마커 (tombstone) 보관 시간 → repair 윈도우.

---

## 9. 디스크 / OS

### 9.1 디스크
- **SSD / NVMe**
- 별도 commitlog 디스크 (HDD 도 OK) — sequential write

### 9.2 OS

```bash
sysctl -w net.ipv4.tcp_keepalive_time=60
sysctl -w vm.max_map_count=1048575
sysctl -w vm.swappiness=1

ulimit -n 100000
ulimit -u 32768
ulimit -l unlimited
```

### 9.3 THP

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

---

## 10. 보안

### 10.1 TLS (inter-node + client)

```yaml
server_encryption_options:
  internode_encryption: all
  keystore: conf/keystore.jks
  keystore_password: ...

client_encryption_options:
  enabled: true
  keystore: conf/keystore.jks
  keystore_password: ...
  require_client_auth: true
```

### 10.2 인증
```yaml
authenticator: PasswordAuthenticator
authorizer: CassandraAuthorizer
```

기본 user `cassandra/cassandra` → 즉시 변경.

---

## 11. 운영 권장 — 16GB / 4 vCPU 노드

```yaml
num_tokens: 16
endpoint_snitch: GossipingPropertyFileSnitch
authenticator: PasswordAuthenticator
authorizer: CassandraAuthorizer

concurrent_reads: 32
concurrent_writes: 32
compaction_throughput_mb_per_sec: 64

read_request_timeout_in_ms: 5000
write_request_timeout_in_ms: 2000
```

```
# jvm-server.options
-Xms8G
-Xmx8G
-XX:+UseG1GC
-XX:MaxGCPauseMillis=300
```

---

## 12. 모니터링

```bash
nodetool status
nodetool info
nodetool tablestats keyspace.table
nodetool tpstats
nodetool compactionstats
nodetool getendpoints keyspace table key
nodetool gossipinfo
```

JMX / Prometheus jmx_exporter / DataDog 통합.

---

## 13. 함정

### 13.1 단일 노드 운영
의미 X. 최소 3 노드.

### 13.2 RF=1
가용성 X. RF=3 표준.

### 13.3 Heap > 32 GB (Java)
GC pause 폭증. ScyllaDB 로 큰 메모리.

### 13.4 seed 모든 노드
seed = 부트스트랩용 만. 2-3 개만.

### 13.5 num_tokens 256 (옛 기본)
shard 마이그 / rebalance 비용 ↑. 16 권장.

### 13.6 SimpleStrategy 운영
NetworkTopologyStrategy 사용.

### 13.7 SimpleSnitch
GossipingPropertyFileSnitch.

### 13.8 Default user `cassandra` 유지
즉시 새 superuser + cassandra 권한 제거.

---

## 14. 학습 자료

- **cassandra.yaml Reference**
- **Cassandra Operations** — DataStax docs
- **The Last Pickle Blog**

---

## 15. 관련

- [[replication-strategy]] — RF / NTS
- [[consistency-tunable]] — CL
- [[cassandra]] — Cassandra hub
