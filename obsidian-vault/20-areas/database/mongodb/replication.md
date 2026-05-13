---
title: "MongoDB Replication — Replica Set"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:00:00+09:00
tags:
  - database
  - mongodb
  - replication
---

# MongoDB Replication — Replica Set

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | RS 구성 / 선출 / Read Preference |

**[[mongodb|↑ MongoDB hub]]**

---

## 1. Replica Set 한 줄

3+ 멤버 (Primary 1 + Secondary N) 의 그룹. Primary 가 모든 쓰기 → oplog → Secondary 들이 적용. Raft 유사 선출.

---

## 2. 구성

### 2.1 mongod.conf

```yaml
replication:
  replSetName: rs0
  oplogSizeMB: 51200

security:
  authorization: enabled
  keyFile: /etc/mongo/rs.key
```

### 2.2 초기화

```js
// 한 노드에서 (admin 으로)
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

### 2.3 멤버 추가 / 제거

```js
rs.add("mongo4:27017")
rs.add({ host: "mongo5:27017", priority: 0, hidden: true })
rs.remove("mongo5:27017")
```

---

## 3. 멤버 타입

| 타입 | 역할 |
| --- | --- |
| **Primary** | 모든 write |
| **Secondary** | replica, read 가능 |
| **Arbiter** | 투표만, 데이터 X (권장 X) |
| **Hidden** | read 안 함, 백업 / 분석용 |
| **Delayed** | N 초 늦게 replay — 복구용 |
| **Priority 0** | primary 후보 X |
| **Voting / Non-voting** | 선출 투표 권한 |

### 3.1 옵션

```js
rs.add({
  host: "backup-node:27017",
  priority: 0,
  hidden: true,
  slaveDelay: 3600          // 1 시간 늦게 (4.0+) — secondaryDelaySecs (5.0+)
})
```

---

## 4. 선출 (Election)

Primary 가 down 되면:
1. Secondary 들이 timeout 감지 (`electionTimeoutMillis` 기본 10초)
2. 후보가 vote 요청
3. 과반수 vote 얻으면 primary 승격
4. Oplog 차이 / priority / wall-clock 으로 우선순위

⚠️ **홀수 멤버** 권장 (과반수 결정).

---

## 5. Oplog

`local.oplog.rs` 라는 capped collection.
- 모든 write 가 idempotent 형태로 기록
- Secondary 가 tail
- 크기가 클수록 secondary 가 끊겨도 따라잡기 쉬움

```js
db.printReplicationInfo()
// "log size", "first event time", "now" 차이 = oplog window
```

```yaml
replication:
  oplogSizeMB: 51200   # 50 GB
```

---

## 6. Read Preference

```js
db.users.find().readPref("secondaryPreferred")

// 옵션
"primary"            // 기본
"primaryPreferred"
"secondary"
"secondaryPreferred"
"nearest"

// tag set
readPref("secondary", [{ region: "kr" }])

// max staleness (4.0+)
{ readPreference: { mode: "secondary", maxStalenessSeconds: 90 } }
```

### 6.1 언제 어디서

- **primary** — 일관성 (read after write)
- **secondaryPreferred** — 읽기 분산, 약간 stale OK
- **nearest** — 글로벌 분산

⚠️ Secondary read 는 **약간 늦은** 데이터. 일관성 중요하면 primary.

---

## 7. Read Concern

| Level | 의미 |
| --- | --- |
| `local` (기본) | 노드 로컬 |
| `available` | 빠른 read (sharded) |
| `majority` | 과반수 commit |
| `linearizable` | wall-clock 직렬 (primary 만, 느림) |
| `snapshot` | 트랜잭션 안 |

자세히 → [[transactions]]

---

## 8. Write Concern

```js
db.users.insertOne(doc, {
  writeConcern: { w: "majority", j: true, wtimeout: 5000 }
})
```

| w | 의미 |
| --- | --- |
| `0` | ack X — fire-and-forget |
| `1` | primary ack |
| `2`, `3`, ... | N 노드 ack |
| `"majority"` (기본 4.4+) | 과반수 ack |

| j | 의미 |
| --- | --- |
| `false` | 메모리 ack |
| `true` | journal flush |

`wtimeout` 안에 충족 안 되면 에러 (단, 작업은 진행).

---

## 9. Causal Consistency

같은 세션 안에서 자기 변경을 다음 read 가 봄 — secondary 에서도.

```js
const session = client.startSession({ causalConsistency: true })
const coll = client.db("app").collection("users", { session })

await coll.insertOne({ _id: 1 }, { session })
const doc = await coll.findOne({ _id: 1 }, { session })   // 봄
```

---

## 10. Failover

```
1. Primary down
2. Secondary 가 감지 (10s)
3. 선출 → 새 primary
4. 클라이언트가 reconnect (드라이버 자동)
```

### 10.1 Stepdown — 수동 swap

```js
rs.stepDown(60)        // 60 초 동안 primary 자격 X
```

업그레이드 / 점검 시.

---

## 11. 모니터링

```js
rs.status()                                  // 멤버 상태
rs.printReplicationInfo()                    // oplog window
rs.printSecondaryReplicationInfo()           // secondary lag
db.serverStatus().repl                       // 세부
db.adminCommand({ replSetGetStatus: 1 })
```

핵심:
- `health: 1`
- `state: PRIMARY | SECONDARY | ARBITER`
- `optimeDate` — 마지막 적용 시간
- `lastHeartbeat`

### 11.1 Lag 알람

```js
rs.printSecondaryReplicationInfo()
// "0 secs behind primary" 표시
```

장기 lag 면 네트워크 / oplog / 부하 점검.

---

## 12. Sharded Cluster + Replication

각 shard 는 **자체 Replica Set**.
Config server 도 Replica Set.
즉, Replication = Sharding 의 토대.

자세히 → [[sharding]]

---

## 13. 백업과 Replication

| 방법 | 일관성 | 권장 |
| --- | --- | --- |
| `mongodump` | local | 작은 DB |
| **Hidden / Delayed secondary** | 시점 일관 | 운영 |
| **Atlas Snapshot** | 시점 + binlog | 매니지드 |
| **filesystem snapshot** | 일관 (정지 후) | 큰 DB |

Hidden/Delayed 노드에서 `mongodump` 가 운영 표준.

---

## 14. 함정

### 함정 1 — Arbiter 의존
투표만 — 데이터 X. write concern 보호 X. 멀티 데이터 멤버 권장.

### 함정 2 — 짝수 멤버
선출 교착 가능. 홀수 (3, 5, 7).

### 함정 3 — Secondary 직접 쓰기
허용 X. 명시적 connection 으로 secondary read 만.

### 함정 4 — Oplog 작음
Secondary 끊긴 후 따라잡기 실패 → full resync. 크게.

### 함정 5 — Read Preference `primary` 만
읽기 분산 X. secondary 사용 검토.

### 함정 6 — Write Concern `w:0` 운영
데이터 손실 발견 X. 최소 `w:1`.

### 함정 7 — `slaveDelay` 인 줄 알았는데 `secondaryDelaySecs`
5.0+ 부터 이름 변경.

### 함정 8 — keyFile 의 권한
0400 + mongodb 소유 아니면 시작 X.

---

## 15. 학습 자료

- **MongoDB Replication** — docs.mongodb.com/manual/replication
- **MongoDB University** — M103 (Basic Cluster Administration)

---

## 16. 관련

- [[sharding]] — Sharded Cluster
- [[transactions]] — Read/Write Concern
- [[configuration]] — RS 설정
- [[mongodb]] — MongoDB hub
