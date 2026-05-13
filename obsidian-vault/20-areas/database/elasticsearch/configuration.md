---
title: "Elasticsearch 설정 — elasticsearch.yml / JVM / 디스크"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:30:00+09:00
tags:
  - database
  - elasticsearch
  - configuration
---

# Elasticsearch 설정

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | yml / JVM / OS |

**[[elasticsearch|↑ Elasticsearch hub]]**

---

## 1. 설정 파일

```
config/elasticsearch.yml
config/jvm.options
config/log4j2.properties
config/users
```

위치: `$ES_HOME/config` 또는 `/etc/elasticsearch/`.

---

## 2. 핵심 설정 — elasticsearch.yml

```yaml
# 클러스터 / 노드
cluster.name: prod-cluster
node.name: ${HOSTNAME}
node.roles: [ master, data, ingest ]

# 데이터 경로
path.data: /var/lib/elasticsearch/data
path.logs: /var/log/elasticsearch

# 네트워크
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300

# Discovery (multi-node)
discovery.seed_hosts: [ "es01", "es02", "es03" ]
cluster.initial_master_nodes: [ "es01", "es02", "es03" ]

# 보안
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/transport.p12
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/http.p12

# 메모리 잠금 (swap 방지)
bootstrap.memory_lock: true
```

---

## 3. 노드 역할 (node.roles)

| Role | 의미 |
| --- | --- |
| `master` | 클러스터 상태 관리 |
| `data` | 데이터 보관 (모든 tier) |
| `data_hot` | hot tier (최신, SSD) |
| `data_warm` | warm tier |
| `data_cold` | cold tier (HDD) |
| `data_frozen` | frozen (object storage) |
| `ingest` | ingest pipeline |
| `ml` | machine learning |
| `transform` | transform |
| `remote_cluster_client` | CCS |
| `coordinating only` | 검색 라우팅만 (`[]`) |

### 3.1 전용 master 노드 (큰 클러스터)
3+ master-only 노드를 별도로. 데이터 / 검색 부담 격리.

---

## 4. JVM — jvm.options

```
-Xms16g
-Xmx16g

# G1 GC (기본 7.x+)
-XX:+UseG1GC
-XX:G1ReservePercent=25
-XX:InitiatingHeapOccupancyPercent=30
```

### 4.1 Heap 크기

- **RAM 의 50%**
- **32 GB 이하** (compressed oops)
- min = max (resize 안 함)
- 나머지 RAM = Lucene 의 OS 파일 캐시 (매우 중요)

64GB 머신:
```
heap = 32g (또는 30g)
나머지 32g = OS 캐시 (Lucene)
```

### 4.2 -XX:+HeapDumpOnOutOfMemoryError
OOM 시 dump.

---

## 5. OS 튜닝

### 5.1 ulimit

```bash
# /etc/security/limits.conf
elasticsearch  -  nofile  65535
elasticsearch  -  memlock unlimited
elasticsearch  -  nproc   4096
```

### 5.2 sysctl

```bash
# /etc/sysctl.conf
vm.max_map_count=262144     # mmap 파일 수
vm.swappiness=1
fs.file-max=1500000
```

### 5.3 swap 비활성

```bash
swapoff -a
# /etc/fstab 의 swap 라인 주석
```

또는 `bootstrap.memory_lock: true` (memlock unlimited 필요).

### 5.4 THP

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

---

## 6. 디스크 / 파일 시스템

- **SSD / NVMe** — Lucene 의 random read 가 핵심
- ext4 / xfs
- mount option `noatime`
- 1 노드 1 데이터 디스크 권장 (multi-path 는 일부 함정)

### 6.1 watermark

```yaml
cluster.routing.allocation.disk.watermark.low: 85%
cluster.routing.allocation.disk.watermark.high: 90%
cluster.routing.allocation.disk.watermark.flood_stage: 95%
```

low 도달 → 새 shard 할당 X.
high → 기존 shard 다른 노드로 이동.
flood → 모든 인덱스 read-only.

---

## 7. 클러스터 상태 / 셋팅 (Dynamic)

```http
GET /_cluster/settings
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.enable": "all"
  },
  "persistent": {
    "indices.recovery.max_bytes_per_sec": "100mb"
  }
}

PUT /_cluster/settings
{ "persistent": { "cluster.routing.allocation.disk.watermark.high": "90%" } }
```

| 종류 | 의미 |
| --- | --- |
| `transient` | 재시작 시 사라짐 |
| `persistent` | 영구 |

---

## 8. Index Settings

```http
PUT /products/_settings
{
  "index": {
    "number_of_replicas": 2,
    "refresh_interval": "5s"
  }
}

GET /products/_settings
```

| Setting | 의미 |
| --- | --- |
| `number_of_shards` | 생성 시 고정 (변경 X) |
| `number_of_replicas` | 동적 변경 |
| `refresh_interval` | 색인 후 검색 가능 시간 (1s 기본) |
| `index.codec` | best_compression / default |
| `analysis.analyzer.*` | analyzer 정의 |
| `index.blocks.write` | true 면 쓰기 차단 |

---

## 9. ILM (Index Lifecycle Management)

```http
PUT /_ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot":  { "actions": { "rollover": { "max_size": "50gb", "max_age": "7d" } } },
      "warm": { "min_age": "7d",  "actions": { "shrink": { "number_of_shards": 1 } } },
      "cold": { "min_age": "30d", "actions": { "freeze": {} } },
      "delete": { "min_age": "90d", "actions": { "delete": {} } }
    }
  }
}

PUT /_index_template/logs
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "index.lifecycle.name": "logs-policy",
      "index.lifecycle.rollover_alias": "logs"
    }
  }
}
```

로그 / 시계열 자동 회전 + 보존 관리.

---

## 10. Snapshot Repository (백업)

```http
PUT /_snapshot/my_repo
{
  "type": "s3",
  "settings": {
    "bucket": "my-es-backups",
    "region": "ap-northeast-2"
  }
}

PUT /_snapshot/my_repo/snap-2026-05-13
{
  "indices": "logs-*",
  "include_global_state": false
}

POST /_snapshot/my_repo/snap-2026-05-13/_restore
{
  "indices": "logs-2026-05",
  "rename_pattern": "logs-(.+)",
  "rename_replacement": "restored_logs-$1"
}
```

S3 / GCS / Azure / Shared FS 지원.

---

## 11. Audit Log

```yaml
xpack.security.audit.enabled: true
xpack.security.audit.logfile.events.include: [ access_denied, authentication_failed ]
```

---

## 12. 권장 클러스터 토폴로지

### 12.1 소 (개발 / POC)
- 1 노드 (master + data + ingest)
- single-node discovery

### 12.2 중 (운영)
- 3 master/data 통합 노드 (또는 3 master + N data)
- replica 1+
- snapshot 정기

### 12.3 대 (운영 대규모)
- 3 전용 master
- N data (hot/warm/cold 분리)
- 전용 ingest / ml / coordinating
- ILM + snapshot

---

## 13. 함정

### 13.1 Heap > 32GB
Compressed OOPS 해제 → 성능 ↓.

### 13.2 Lucene 캐시 부족
heap 50%, 나머지 OS 캐시 → 검색 성능의 핵심.

### 13.3 vm.max_map_count 작음
색인 / merge 시 mmap 부족. 262144+.

### 13.4 swap 사용
GC pause 폭증. swap off 또는 memory_lock.

### 13.5 보안 비활성
8.0+ 는 기본 활성. 끄지 말 것.

### 13.6 단일 노드 + replica 1
replica unassigned. dev 면 0, 운영은 노드 ≥ 2.

### 13.7 cluster.initial_master_nodes
처음 부트스트랩 시에만. 이후 제거 권장 (한 번 적용 후).

### 13.8 단일 master + master.eligible = 1
SPOF. master-eligible 은 홀수 (3, 5).

---

## 14. 학습 자료

- **Elasticsearch Reference** — elastic.co/guide/en/elasticsearch/reference
- **Production Checklist** — elastic.co/guide → important-settings
- **Elastic Stack Best Practices**

---

## 15. 관련

- [[cluster-shards]] — shard / replica
- [[performance-tuning]] — 튜닝
- [[elasticsearch]] — ES hub
