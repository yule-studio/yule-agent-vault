---
title: "MongoDB 트랜잭션 — Multi-document ACID"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:55:00+09:00
tags:
  - database
  - mongodb
  - transaction
---

# MongoDB 트랜잭션

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Multi-document / Replica Set / Sharded |

**[[mongodb|↑ MongoDB hub]]**

---

## 1. 한 줄

- **4.0+** Replica Set 에서 multi-document ACID.
- **4.2+** Sharded Cluster 에서도.
- 단, **단일 document 의 변경은 항상 원자적** — 가능하면 임베드로 표현.

---

## 2. 단일 document 의 원자성 (트랜잭션 없이도)

```js
db.accounts.updateOne(
  { _id: 1 },
  { $inc: { balance: -100, savings: +100 } }
)
```

→ 한 document 안의 모든 필드 변경은 원자적. 임베드 모델이 트랜잭션 없이 많은 케이스를 처리.

---

## 3. Multi-document Transaction

```js
const session = client.startSession()
try {
  session.startTransaction({
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority" }
  })
  
  await db.accounts.updateOne(
    { _id: 1 }, { $inc: { balance: -100 } },
    { session }
  )
  await db.accounts.updateOne(
    { _id: 2 }, { $inc: { balance: +100 } },
    { session }
  )
  
  await session.commitTransaction()
} catch (err) {
  await session.abortTransaction()
  throw err
} finally {
  session.endSession()
}
```

### 3.1 mongosh

```js
const s = db.getMongo().startSession()
s.startTransaction()
s.getDatabase("app").accounts.updateOne({_id:1}, {$inc:{balance:-100}})
s.getDatabase("app").accounts.updateOne({_id:2}, {$inc:{balance:+100}})
s.commitTransaction()
s.endSession()
```

---

## 4. 옵션

### 4.1 readConcern

| Level | 의미 |
| --- | --- |
| `local` | 로컬 |
| `majority` | 과반수 commit |
| `snapshot` | 트랜잭션의 일관 snapshot (기본) |

### 4.2 writeConcern

| 옵션 | 의미 |
| --- | --- |
| `w: 1` | primary ack |
| `w: "majority"` (기본) | 과반수 |
| `j: true` | journal flush |
| `wtimeout: 5000` | 초과 시 에러 |

---

## 5. 재시도 가능 에러

MongoDB 는 transient transaction error 시 재시도 권장. 라이브러리가 `TransientTransactionError` / `UnknownTransactionCommitResult` 라벨.

```js
async function runWithRetry(txnFn) {
  while (true) {
    try {
      return await txnFn()
    } catch (e) {
      if (e.hasErrorLabel("TransientTransactionError")) continue
      if (e.hasErrorLabel("UnknownTransactionCommitResult")) continue
      throw e
    }
  }
}
```

---

## 6. 제약 / 한계

| 항목 | 한계 |
| --- | --- |
| 트랜잭션 시간 | 기본 60 초 (`transactionLifetimeLimitSeconds`) |
| Document 크기 | 트랜잭션 중 변경 16 MB 한계 |
| Oplog 크기 | 트랜잭션 변경이 oplog 안 |
| Collection 생성 | 4.4+ 부터 트랜잭션 안에서 가능 |
| `count` | 사용 X — `countDocuments` 도 일부 X |
| Multi-shard 트랜잭션 | 4.2+ — 비용 ↑ |

---

## 7. 성능 vs 임베드

트랜잭션은 **비싸다**:
- snapshot isolation 유지
- replica set commit
- 2-phase commit (sharded)

→ 임베드로 단일 document 변경이 가능하면 **그게 표준**.

```js
// ❌ 트랜잭션
// users 와 settings 가 분리
session.startTransaction()
db.users.updateOne({_id:1}, {$set:{name:"B"}}, {session})
db.settings.updateOne({userId:1}, {$set:{theme:"dark"}}, {session})
session.commitTransaction()

// ✅ 임베드
db.users.updateOne(
  {_id:1},
  {$set: {name: "B", "settings.theme": "dark"}}
)
```

---

## 8. WiredTiger MVCC

스토리지 엔진의 MVCC → 읽기-쓰기 lock-free.
PostgreSQL 의 MVCC 와 비슷한 개념. 트랜잭션은 그 위에 구축.

---

## 9. 읽기 일관성

### 9.1 Read Concern 별 동작

| Level | 보장 |
| --- | --- |
| `local` | 노드 최신 (replica 도 자기 시점) |
| `available` | sharded 의 빠른 read |
| `majority` | 과반수 commit 된 것만 |
| `snapshot` | 트랜잭션 시점 일관 |
| `linearizable` | wall-clock 시간 순 (느림, primary 만) |

### 9.2 Causal Consistency

```js
const session = client.startSession({ causalConsistency: true })
const db = client.db("app", { session })

await db.users.insertOne({_id:1, name:"A"})
const found = await db.users.findOne({_id:1})  // 항상 자기 변경 봄
```

같은 세션 안에서 자기 변경 후 read 가 그것을 보장.

---

## 10. Sharded Cluster 트랜잭션

4.2+ 에서 cross-shard 트랜잭션. 2-phase commit.

```js
// 자동 처리됨 — 클라이언트 코드 동일
session.startTransaction()
db.users.updateOne({shardKey:"A"}, ...)
db.orders.updateOne({shardKey:"B"}, ...)
session.commitTransaction()
```

비용 ↑ — 단일 shard 안에서 끝나도록 shard key 설계.

---

## 11. 실전 패턴

### 11.1 송금

```js
async function transfer(from, to, amount) {
  return await runWithRetry(async () => {
    const session = client.startSession()
    try {
      session.startTransaction()
      const acc = await db.accounts.findOne({_id: from}, {session})
      if (acc.balance < amount) throw new Error("부족")
      
      await db.accounts.updateOne({_id: from}, {$inc:{balance:-amount}}, {session})
      await db.accounts.updateOne({_id: to},   {$inc:{balance:+amount}}, {session})
      await db.transactions.insertOne({from, to, amount, ts: new Date()}, {session})
      
      await session.commitTransaction()
    } catch (e) {
      await session.abortTransaction()
      throw e
    } finally {
      session.endSession()
    }
  })
}
```

### 11.2 분산 트랜잭션 대안 — Saga

트랜잭션이 비싸거나 cross-service 면 Saga (compensating action).

---

## 12. 함정

### 함정 1 — 단일 document 의 트랜잭션
필요 X. 자동 원자.

### 함정 2 — 긴 트랜잭션
60초 기본 limit. 짧게.

### 함정 3 — 트랜잭션 안의 비싼 쿼리
다른 트랜잭션 대기. write conflict.

### 함정 4 — 재시도 누락
TransientTransactionError 에 재시도 필요.

### 함정 5 — Sharded 트랜잭션 남용
2PC 비용. single-shard 로 설계.

### 함정 6 — `count`
일부 메소드 트랜잭션 안에서 미지원.

### 함정 7 — Collection 자동 생성
4.4 이전엔 트랜잭션 안에서 컬렉션 자동 생성 X — 미리 생성.

### 함정 8 — Read Preference `primary` 강제
트랜잭션은 primary 만. secondary 우선이면 동작 X.

---

## 13. 학습 자료

- **MongoDB Transactions** — docs.mongodb.com/manual/core/transactions
- **MongoDB University** — M201
- **Distributed Transactions in MongoDB** — blog post

---

## 14. 관련

- [[data-modeling]] — 임베드로 트랜잭션 회피
- [[replication]] — write concern
- [[mongodb]] — MongoDB hub
