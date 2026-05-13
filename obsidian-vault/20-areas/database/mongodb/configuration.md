---
title: "MongoDB 설정 — mongod.conf"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:30:00+09:00
tags:
  - database
  - mongodb
  - configuration
---

# MongoDB 설정 — mongod.conf

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 핵심 설정 + 보안 + 리소스 |

**[[mongodb|↑ MongoDB hub]]**

---

## 1. 설정 파일

```
/etc/mongod.conf                  # Linux 표준
/usr/local/etc/mongod.conf        # macOS brew
```

YAML 포맷.

---

## 2. 표본 구조

```yaml
storage:
  dbPath: /var/lib/mongodb
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen

net:
  port: 27017
  bindIp: 0.0.0.0
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongo/server.pem
    CAFile: /etc/mongo/ca.pem

security:
  authorization: enabled
  keyFile: /etc/mongo/replicaset.key    # replica set 내부

replication:
  replSetName: rs0

sharding:
  clusterRole: shardsvr               # mongos / configsvr / shardsvr

processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid
```

---

## 3. 메모리 — WiredTiger Cache

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8
      directoryForIndexes: false
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true
```

### 3.1 cacheSizeGB

기본: `max(0.5 * (RAM - 1GB), 256MB)`.
권장: **RAM 의 50%** 정도. 너무 크면 OS 캐시 압박.

### 3.2 압축

| Compressor | 특성 |
| --- | --- |
| `none` | 무압축 |
| `snappy` (기본) | 빠름 + 적당 |
| `zlib` | 더 작음, CPU ↑ |
| `zstd` (4.2+) | 균형 |

---

## 4. 보안

### 4.1 인증

```yaml
security:
  authorization: enabled
```

⚠️ 운영 필수. 처음엔 admin 사용자 만들고 활성.

### 4.2 Replica Set 내부 — keyFile / x509

```yaml
security:
  authorization: enabled
  keyFile: /etc/mongo/rs.key
```

```bash
# rs.key 생성 (6 자~1024 자, 모든 노드 동일)
openssl rand -base64 756 > rs.key
chmod 400 rs.key
chown mongodb:mongodb rs.key
```

x509 인증서 권장 (강함).

### 4.3 TLS

```yaml
net:
  tls:
    mode: requireTLS         # disabled / allowTLS / preferTLS / requireTLS
    certificateKeyFile: /etc/mongo/server.pem
    CAFile: /etc/mongo/ca.pem
    allowConnectionsWithoutCertificates: false
```

### 4.4 bindIp

```yaml
net:
  bindIp: 127.0.0.1            # 로컬만
  bindIp: 0.0.0.0              # 모두 (방화벽 + 인증 필수)
  bindIpAll: true              # 모든 인터페이스
```

---

## 5. 로그

```yaml
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen
  verbosity: 0              # 0-5
  component:
    query:
      verbosity: 1          # 컴포넌트별 verbosity
```

### 5.1 Profiler

```js
db.setProfilingLevel(1, { slowms: 100 })
//  0 = off
//  1 = slow only (> slowms)
//  2 = all

db.system.profile.find().sort({ ts: -1 }).limit(10)
```

---

## 6. 리소스 / 운영체제

```yaml
operationProfiling:
  slowOpThresholdMs: 100
  mode: slowOp

setParameter:
  diagnosticDataCollectionEnabled: true
  internalQueryExecMaxBlockingSortBytes: 33554432
```

### 6.1 OS

```bash
# THP 끔 (필수)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# ulimit
ulimit -n 64000

# swap 최소화
sysctl vm.swappiness=1
```

systemd:
```ini
[Service]
LimitFSIZE=infinity
LimitCPU=infinity
LimitAS=infinity
LimitNOFILE=64000
LimitNPROC=64000
LimitMEMLOCK=infinity
```

---

## 7. Replica Set 설정

```yaml
replication:
  replSetName: rs0
  oplogSizeMB: 51200          # 50 GB
```

```js
// 초기화 (한 노드에서)
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017" },
    { _id: 1, host: "mongo2:27017" },
    { _id: 2, host: "mongo3:27017" }
  ]
})

rs.status()
rs.conf()
```

자세히 → [[replication]]

---

## 8. Sharding 설정

| 역할 | clusterRole |
| --- | --- |
| Shard server | `shardsvr` |
| Config server | `configsvr` |
| Router (mongos) | (별도 binary, mongos) |

```yaml
sharding:
  clusterRole: shardsvr

replication:
  replSetName: shard1
```

자세히 → [[sharding]]

---

## 9. Connection 제한

```yaml
net:
  maxIncomingConnections: 65536
```

기본 65536. 실제 클라이언트 풀과 OS ulimit 보다 큰 값.

---

## 10. 권장 시작 — 16GB / 4 vCPU

```yaml
storage:
  dbPath: /var/lib/mongodb
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: snappy

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

net:
  port: 27017
  bindIp: 0.0.0.0
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongo/server.pem

security:
  authorization: enabled
  keyFile: /etc/mongo/rs.key

replication:
  replSetName: rs0
  oplogSizeMB: 51200

operationProfiling:
  slowOpThresholdMs: 100
  mode: slowOp
```

---

## 11. Schema Validation (선택)

```js
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "createdAt"],
      properties: {
        email: { bsonType: "string", pattern: "^.+@.+$" },
        age:   { bsonType: "int", minimum: 0, maximum: 150 }
      }
    }
  },
  validationLevel: "moderate",
  validationAction: "error"
})
```

---

## 12. 함정

### 함정 1 — `authorization: disabled`
사고. 항상 enabled.

### 함정 2 — `cacheSizeGB` 너무 큼
OS 캐시 압박 / OOM. RAM 50%.

### 함정 3 — `bindIp: 0.0.0.0` + 인증 없음
즉시 침해.

### 함정 4 — THP on
Latency spike. 끔.

### 함정 5 — Replica Set 미구성
단일 노드 = 트랜잭션 X, failover X.

### 함정 6 — Single member RS
가용성 향상 X. 최소 3 멤버 (또는 P+S+arbiter, 단 arbiter 권장 X).

### 함정 7 — oplogSize 너무 작음
replica 가 끊긴 후 따라잡기 실패 → full resync. 충분히 크게.

### 함정 8 — keyFile 권한
0400 mongodb 소유 아니면 시작 안 됨.

---

## 13. 학습 자료

- **MongoDB Configuration File Options** — docs.mongodb.com
- **Production Notes** — Operating System Recommendations
- **MongoDB University** — DBA path

---

## 14. 관련

- [[replication]]
- [[sharding]]
- [[mongodb]] — MongoDB hub
