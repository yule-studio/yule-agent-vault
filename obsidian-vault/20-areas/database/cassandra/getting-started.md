---
title: "Cassandra 시작하기 — 설치 / cqlsh / 첫 명령"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:05:00+09:00
tags:
  - database
  - cassandra
  - setup
---

# Cassandra 시작하기

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 설치 / cqlsh / 첫 명령 |

**[[cassandra|↑ Cassandra hub]]**

---

## 1. 설치

### 1.1 Docker (가장 빠름)

```bash
docker run -d \
  --name cas \
  -p 9042:9042 \
  -e CASSANDRA_CLUSTER_NAME=dev \
  cassandra:5.0

# ScyllaDB
docker run -d \
  --name scylla \
  -p 9042:9042 \
  scylladb/scylla --developer-mode 1
```

### 1.2 Ubuntu

```bash
echo "deb https://debian.cassandra.apache.org 50x main" | sudo tee /etc/apt/sources.list.d/cassandra.list
curl https://downloads.apache.org/cassandra/KEYS | sudo apt-key add -
sudo apt update
sudo apt install cassandra

sudo systemctl enable --now cassandra
```

### 1.3 매니지드

| 서비스 | 특징 |
| --- | --- |
| **AWS Keyspaces** | Cassandra 호환 매니지드 |
| **Astra DB (DataStax)** | 서버리스 Cassandra |
| **ScyllaDB Cloud** | Scylla 매니지드 |
| **Instaclustr** | 멀티 클라우드 |

---

## 2. cqlsh — Shell

```bash
cqlsh                              # localhost:9042
cqlsh host 9042 -u cassandra -p ...
cqlsh --ssl
```

기본 superuser: `cassandra` / `cassandra` (활성 시).

---

## 3. Keyspace 와 Table

Keyspace = RDB 의 database / schema.

```sql
-- Keyspace 생성
CREATE KEYSPACE app
WITH REPLICATION = {
  'class': 'NetworkTopologyStrategy',
  'datacenter1': 3
};

USE app;

-- 테이블
CREATE TABLE users (
  user_id UUID PRIMARY KEY,
  email   TEXT,
  name    TEXT,
  created_at TIMESTAMP
);

-- 시계열 패턴 (partition + clustering)
CREATE TABLE events (
  user_id UUID,
  event_time TIMESTAMP,
  event_type TEXT,
  data TEXT,
  PRIMARY KEY ((user_id), event_time)
) WITH CLUSTERING ORDER BY (event_time DESC);
```

### 3.1 Primary Key 구조

```
PRIMARY KEY ((partition_key), clustering_key1, clustering_key2)
```

- **partition_key** — 같은 노드에 저장 단위 (`(...)` 안)
- **clustering_key** — partition 안의 정렬

---

## 4. INSERT / UPDATE / DELETE

```sql
INSERT INTO users (user_id, email, name)
VALUES (uuid(), 'a@x.com', 'A');

INSERT INTO users (user_id, email, name)
VALUES (uuid(), 'b@x.com', 'B')
USING TTL 86400;                    -- 24시간 후 자동 삭제

UPDATE users SET name = 'B2' WHERE user_id = 550e...;

DELETE FROM users WHERE user_id = 550e...;
DELETE name FROM users WHERE user_id = 550e...;   -- 컬럼만 NULL 로
```

⚠️ INSERT 와 UPDATE 는 거의 같음 — 둘 다 upsert.

---

## 5. SELECT — partition key 가 필수

```sql
SELECT * FROM users WHERE user_id = 550e...;

-- ✅ partition key 포함
SELECT * FROM events WHERE user_id = 550e...
  AND event_time > '2026-05-01'
  ORDER BY event_time DESC LIMIT 100;

-- ❌ partition key 없음 — ALLOW FILTERING 필요 (위험)
SELECT * FROM events WHERE event_type = 'login' ALLOW FILTERING;
```

`ALLOW FILTERING` = 모든 노드 scan. **운영 X**.

---

## 6. cqlsh 메타 명령

```
DESCRIBE KEYSPACES;
DESC KEYSPACE app;
DESC TABLES;
DESC TABLE users;
SHOW SESSION;
SHOW VERSION;
EXPAND ON;            -- 세로 출력
TRACING ON;           -- 쿼리 trace
PAGING 100;           -- 페이지 크기

SOURCE 'script.cql';
COPY users TO 'users.csv' WITH HEADER = TRUE;
COPY users FROM 'users.csv' WITH HEADER = TRUE;
```

---

## 7. nodetool — 운영 CLI

```bash
nodetool status                    # 모든 노드 상태
nodetool info                      # 현재 노드 정보
nodetool ring                      # token ring
nodetool tablestats app            # 테이블 통계
nodetool tpstats                   # thread pool
nodetool compactionstats
nodetool flush app users           # memtable → SSTable
nodetool repair app                # anti-entropy 복구
nodetool cleanup                   # 다른 노드의 데이터 정리
nodetool drain                     # 종료 전
nodetool snapshot                  # 백업 snapshot
```

---

## 8. 인증 / 권한

```sql
-- 활성 (cassandra.yaml)
--   authenticator: PasswordAuthenticator
--   authorizer: CassandraAuthorizer

CREATE ROLE alice WITH PASSWORD = 'secret' AND LOGIN = true;
GRANT SELECT ON KEYSPACE app TO alice;
GRANT MODIFY ON app.users TO alice;
GRANT EXECUTE ON FUNCTION app.func TO alice;

LIST ALL PERMISSIONS OF alice;

ALTER ROLE alice WITH PASSWORD = 'new';
DROP ROLE alice;
```

---

## 9. 클라이언트 라이브러리

| 언어 | 라이브러리 |
| --- | --- |
| Python | `cassandra-driver` |
| Java | `cassandra-driver-core` / DataStax Java Driver |
| Go | `gocql` |
| Node | `cassandra-driver` (DataStax) |
| Rust | `scylla` (ScyllaDB official, Cassandra 호환) |

```python
from cassandra.cluster import Cluster
from cassandra.auth import PlainTextAuthProvider

auth = PlainTextAuthProvider(username='alice', password='secret')
cluster = Cluster(['node1','node2','node3'], port=9042, auth_provider=auth)
session = cluster.connect('app')

# Prepared statement (필수)
stmt = session.prepare("SELECT * FROM users WHERE user_id = ?")
rows = session.execute(stmt, [user_id])
```

### 9.1 Prepared Statement 필수
같은 쿼리는 prepare 후 execute. 그래야 효율 + 안전.

---

## 10. 토폴로지

최소 운영 구성:
- **3 노드** + RF=3 + Quorum (2/3)
- 단일 DC

다중 DC:
- DC1: 3 노드, RF=3
- DC2: 3 노드, RF=3
- LOCAL_QUORUM 사용

---

## 11. 함정

### 11.1 ALLOW FILTERING
운영 사용 금지. 데이터 모델 재설계.

### 11.2 단일 노드 / RF=1
가용성 X. 최소 3 노드.

### 11.3 cqlsh password 평문
TLS / 환경변수 권장.

### 11.4 9042 외부 노출
보안 / 방화벽.

### 11.5 nodetool 명령 무지
운영의 표준 도구. 익숙해질 것.

### 11.6 partition key 누락
운영 시 ALLOW FILTERING 만나면 데이터 모델 잘못.

---

## 12. 관련

- [[configuration]] — cassandra.yaml
- [[data-modeling]] — partition / clustering
- [[cql-syntax]] — CQL
- [[cassandra]] — Cassandra hub
