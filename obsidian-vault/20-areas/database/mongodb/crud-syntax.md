---
title: "MongoDB CRUD 문법 — find / insert / update / delete / 연산자"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:40:00+09:00
tags:
  - database
  - mongodb
  - crud
---

# MongoDB CRUD 문법

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | find / update / 연산자 |

**[[mongodb|↑ MongoDB hub]]**

---

## 1. Insert

```js
db.users.insertOne({ name: "Alice", age: 30 })
db.users.insertMany([{ ... }, { ... }])

// ordered: false — 중간 실패해도 계속
db.users.insertMany([...], { ordered: false })
```

---

## 2. Find — 기본

```js
db.users.find()                                  // 전체
db.users.find({ age: 30 })                       // 조건
db.users.findOne({ email: "a@x.com" })           // 1 개
db.users.find().pretty()
db.users.find().count()                          // 비효율 (사용 X)
db.users.countDocuments({ age: { $gte: 18 } })   // 권장
```

### 2.1 Projection — 컬럼 선택

```js
db.users.find({}, { name: 1, email: 1 })         // 포함
db.users.find({}, { _id: 0, name: 1 })           // _id 제외
db.users.find({}, { password: 0 })               // 제외
```

### 2.2 Sort / Limit / Skip

```js
db.users.find().sort({ createdAt: -1 }).limit(10)
db.users.find().sort({ age: 1, name: 1 })
db.users.find().skip(20).limit(10)               // ⚠️ 큰 skip 은 느림
```

### 2.3 cursor

```js
const cursor = db.users.find()
while (cursor.hasNext()) {
  const doc = cursor.next()
  ...
}
```

---

## 3. Query Operator (필수)

### 3.1 비교

```js
{ age: { $eq: 30 } }
{ age: { $ne: 30 } }
{ age: { $gt: 18 } }
{ age: { $gte: 18 } }
{ age: { $lt: 65 } }
{ age: { $lte: 65 } }
{ age: { $in: [25, 30, 35] } }
{ age: { $nin: [10, 20] } }
```

### 3.2 논리

```js
{ $and: [{ age: { $gte: 18 } }, { age: { $lt: 65 } }] }
{ $or:  [{ role: "admin" }, { role: "editor" }] }
{ $not: { age: { $gte: 18 } } }
{ $nor: [{ active: true }, { admin: true }] }
```

### 3.3 존재 / 타입

```js
{ email: { $exists: true } }
{ email: { $exists: false } }
{ age:   { $type: "int" } }
{ age:   { $type: ["int", "long"] } }
```

### 3.4 정규식

```js
{ email: /@example\.com$/ }
{ email: { $regex: "^a", $options: "i" } }
```

### 3.5 배열

```js
{ tags: "admin" }                                // 포함
{ tags: { $all: ["admin", "active"] } }          // 모두
{ tags: { $size: 3 } }
{ tags: { $elemMatch: { $regex: "^a" } } }

// 중첩 객체 배열
db.users.find({
  addresses: { $elemMatch: { city: "Seoul", type: "home" } }
})
```

### 3.6 중첩 필드

```js
{ "settings.theme": "dark" }
{ "addresses.0.city": "Seoul" }                  // 첫 element
```

### 3.7 텍스트 (text index 필요)

```js
db.articles.createIndex({ title: "text", body: "text" })
db.articles.find({ $text: { $search: "mongodb tutorial" } })
db.articles.find({ $text: { $search: "\"exact phrase\"" } })
db.articles.find({ $text: { $search: "mongodb -mysql" } })   // 제외
```

### 3.8 Geo

```js
db.places.createIndex({ loc: "2dsphere" })
db.places.find({
  loc: { $near: { $geometry: { type: "Point", coordinates: [127.0, 37.5] }, $maxDistance: 1000 } }
})
db.places.find({
  loc: { $geoWithin: { $centerSphere: [[127.0, 37.5], 1/6378.1] } }   // 1km
})
```

---

## 4. Update

```js
db.users.updateOne(
  { _id: 1 },
  { $set: { age: 31 } }
)

db.users.updateMany(
  { role: "guest" },
  { $set: { active: false } }
)

db.users.replaceOne(
  { _id: 1 },
  { name: "Alice", age: 31 }                     // 통째로 교체
)

// upsert
db.users.updateOne(
  { email: "a@x.com" },
  { $set: { age: 30 }, $setOnInsert: { createdAt: new Date() } },
  { upsert: true }
)

// findOneAndUpdate — 변경 후 반환
db.users.findOneAndUpdate(
  { _id: 1 },
  { $inc: { score: 1 } },
  { returnDocument: "after" }
)
```

---

## 5. Update Operator

### 5.1 필드

```js
$set:    { name: "B" }
$unset:  { tempField: "" }
$rename: { oldName: "newName" }
$inc:    { count: 1, score: 10 }
$mul:    { price: 1.1 }
$min:    { lowestScore: 80 }      // 더 작으면 업데이트
$max:    { highScore: 200 }
$currentDate: { updatedAt: true }
```

### 5.2 배열

```js
$push:     { tags: "new" }
$push:     { tags: { $each: ["a","b"], $slice: -10 } }  // capped 10
$push:     { scores: { $each: [...], $sort: -1, $slice: 5 } }
$addToSet: { tags: "unique" }
$pop:      { stack: 1 }                                  // 1=뒤, -1=앞
$pull:     { tags: "old" }
$pullAll:  { tags: ["a","b"] }
$pull:     { items: { qty: 0 } }                         // 조건
```

### 5.3 배열 위치

```js
// $ — 첫 매칭 위치
db.posts.updateOne(
  { _id: 1, "comments.id": 5 },
  { $set: { "comments.$.text": "updated" } }
)

// $[] — 모든 element
{ $set: { "comments.$[].seen": true } }

// $[<identifier>] — 조건 매칭 element
db.posts.updateOne(
  { _id: 1 },
  { $set: { "comments.$[c].seen": true } },
  { arrayFilters: [{ "c.id": 5 }] }
)
```

---

## 6. Delete

```js
db.users.deleteOne({ _id: 1 })
db.users.deleteMany({ active: false })

db.users.findOneAndDelete({ _id: 1 })

// 컬렉션 / DB 삭제
db.users.drop()
db.dropDatabase()
```

---

## 7. Bulk Write

```js
db.users.bulkWrite([
  { insertOne: { document: { name: "A" } } },
  { updateOne: { filter: { _id: 1 }, update: { $set: { x: 1 } } } },
  { deleteOne: { filter: { _id: 2 } } }
], { ordered: false })
```

성능 ↑ — 라운드트립 1 회.

---

## 8. Read Concern / Write Concern

### 8.1 Write Concern

```js
db.users.insertOne(doc, { writeConcern: { w: "majority", j: true, wtimeout: 5000 } })
```

| 옵션 | 의미 |
| --- | --- |
| `w: 1` | primary ack |
| `w: "majority"` | 과반수 ack |
| `w: 0` | ack 없음 (fire-and-forget) |
| `j: true` | journal flush 까지 |

### 8.2 Read Concern

```js
db.users.find().readConcern("majority")
```

| Level | 의미 |
| --- | --- |
| `local` (기본) | 노드 로컬 |
| `available` | sharded read |
| `majority` | 과반수 commit |
| `linearizable` | 가장 강함 (느림) |
| `snapshot` | 트랜잭션 안 |

### 8.3 Read Preference

```js
db.users.find().readPref("secondaryPreferred")
```

| 모드 | 의미 |
| --- | --- |
| `primary` | primary 만 |
| `primaryPreferred` | primary 우선, 없으면 secondary |
| `secondary` | secondary 만 |
| `secondaryPreferred` | secondary 우선 |
| `nearest` | 가장 가까운 |

자세히 → [[replication]]

---

## 9. 트랜잭션

```js
const session = client.startSession()
try {
  session.startTransaction()
  
  db.accounts.updateOne({_id:1}, {$inc:{balance:-100}}, {session})
  db.accounts.updateOne({_id:2}, {$inc:{balance:+100}}, {session})
  
  session.commitTransaction()
} catch (e) {
  session.abortTransaction()
} finally {
  session.endSession()
}
```

자세히 → [[transactions]]

---

## 10. explain — 실행 계획

```js
db.users.find({ age: 30 }).explain()
db.users.find({ age: 30 }).explain("executionStats")
db.users.find({ age: 30 }).explain("allPlansExecution")

// aggregate
db.users.explain("executionStats").aggregate([...])
```

`COLLSCAN` (전체) / `IXSCAN` (인덱스). 자세히 → [[indexes]]

---

## 11. Aggregation

복잡 쿼리 / GROUP BY / JOIN 은 aggregation pipeline:

```js
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$userId", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } },
  { $limit: 10 }
])
```

자세히 → [[aggregation]]

---

## 12. 함정

### 함정 1 — `count()` 의 비효율
큰 컬렉션에 `countDocuments` (정확) / `estimatedDocumentCount` (메타).

### 함정 2 — 큰 `skip()`
페이지 100,000 = 느림. cursor (`_id` 기반) 사용.

### 함정 3 — `find` 후 `count`
같은 cursor. 분리 호출이면 두 번 실행.

### 함정 4 — 정규식 `^` 없으면 인덱스 X
`/foo/` 는 풀스캔. `/^foo/` 면 인덱스.

### 함정 5 — `$or` + 인덱스
각 항목이 인덱스 있어야 효율. 없으면 풀스캔.

### 함정 6 — `replaceOne` 으로 부분만
전체 교체 — 다른 필드 사라짐.

### 함정 7 — Bulk write `ordered: true`
한 실패에 나머지 중단. 가능하면 `false`.

### 함정 8 — `findAndModify` deprecated
`findOneAndUpdate` / `findOneAndDelete` / `findOneAndReplace` 사용.

---

## 13. 학습 자료

- **MongoDB CRUD Operations** — docs.mongodb.com/manual/crud
- **Query Operators Reference**
- **MongoDB University** — M001/M201

---

## 14. 관련

- [[indexes]] — 인덱스
- [[aggregation]] — Aggregation
- [[transactions]]
- [[mongodb]] — MongoDB hub
