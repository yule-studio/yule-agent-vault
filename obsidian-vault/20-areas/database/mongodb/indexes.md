---
title: "MongoDB 인덱스 — 단일 / 복합 / Text / Geo / Wildcard / TTL"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:45:00+09:00
tags:
  - database
  - mongodb
  - index
---

# MongoDB 인덱스

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 모든 인덱스 종류 |

**[[mongodb|↑ MongoDB hub]]**

---

## 1. 인덱스 종류

| 종류 | 용도 |
| --- | --- |
| Single Field | 단일 필드 |
| Compound | 다중 필드 |
| Multikey | 배열 필드 (자동) |
| Text | 풀텍스트 |
| 2dsphere / 2d | 지리 |
| Hashed | 균등 분포 (sharding) |
| Wildcard | 임의 필드 |
| TTL | 자동 만료 |
| Partial | 조건부 |
| Sparse | NULL 제외 |
| Unique | 유일 |

---

## 2. 생성 / 삭제

```js
db.users.createIndex({ email: 1 })
db.users.createIndex({ email: 1 }, { unique: true })
db.users.createIndex({ email: 1 }, { name: "email_idx" })

db.users.getIndexes()
db.users.dropIndex("email_1")
db.users.dropIndexes()                    // _id 제외 모두
```

### 2.1 방향

```js
{ field: 1 }    // 오름차
{ field: -1 }   // 내림차
```

단일 필드는 둘 다 같음 (B-Tree 양방향). **복합** 에선 의미 있음.

---

## 3. Compound Index

```js
db.orders.createIndex({ userId: 1, createdAt: -1 })
```

### 3.1 왼쪽 접두 규칙

```
인덱스: { a:1, b:1, c:1 }

✅ 사용
{ a: 1 }
{ a: 1, b: 1 }
{ a: 1, b: 1, c: 1 }
{ a: 1, c: 1 }    // 부분 — a 사용, c 는 풀스캔 안의 IXSCAN

❌ 사용 X
{ b: 1 }
{ c: 1 }
```

### 3.2 ESR 규칙 (Equality, Sort, Range)

복합 인덱스 컬럼 순서 = **Equality → Sort → Range**:

```js
// 쿼리
db.orders.find({ userId: 1, status: "pending" })
         .sort({ createdAt: -1 })

// 인덱스
{ userId: 1, status: 1, createdAt: -1 }
//  E       E          S
```

Range 컬럼 (`$gt`, `$lt`) 은 마지막.

### 3.3 정렬 순서

```js
// 인덱스 { a:1, b:-1 }
.sort({ a: 1, b: -1 })    // ✅
.sort({ a: -1, b: 1 })    // ✅ (전체 역순)
.sort({ a: 1, b: 1 })     // ❌
```

---

## 4. Multikey — 배열

```js
db.users.createIndex({ tags: 1 })
// 배열의 각 element 가 인덱스에 들어감

db.users.find({ tags: "admin" })          // ✅
db.users.find({ tags: { $all: ["a","b"] } })
```

### 4.1 한계
- Compound 에서 **하나의 배열 필드만** 가능
- Hashed / Geo 와 조합 일부 제약
- Coverage 제한 (배열 element 를 직접 반환 X)

---

## 5. Text Index

```js
db.articles.createIndex({ title: "text", body: "text" })
db.articles.createIndex({ "$**": "text" })       // 모든 필드

db.articles.find({ $text: { $search: "mongodb tutorial" } })
db.articles.find(
  { $text: { $search: "mongo" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```

### 5.1 옵션

```js
db.articles.createIndex(
  { title: "text", body: "text" },
  {
    weights: { title: 10, body: 1 },
    default_language: "english",
    language_override: "lang"
  }
)
```

### 5.2 한국어
MongoDB text index 는 한국어 형태소 분석 부족 — Atlas Search (Lucene) 또는 Elasticsearch.

### 5.3 한 컬렉션에 text index 1 개

---

## 6. Geo

### 6.1 2dsphere (지구 표면)

```js
db.places.createIndex({ loc: "2dsphere" })

// GeoJSON 형식
{ loc: { type: "Point", coordinates: [127.0, 37.5] } }
{ loc: { type: "Polygon", coordinates: [...] } }

db.places.find({
  loc: { $near: { $geometry: { type: "Point", coordinates: [127, 37.5] }, $maxDistance: 1000 } }
})
```

### 6.2 2d (평면 — 옛)

```js
db.places.createIndex({ loc: "2d" })
{ loc: [127.0, 37.5] }
```

---

## 7. Hashed Index

```js
db.users.createIndex({ _id: "hashed" })
```

- `=` 만 — 범위 X
- Sharding 의 균등 분포에 유용
- 값 자체를 보존 안 함

---

## 8. Wildcard Index (4.2+)

알 수 없는 필드 구조의 schemaless data 에:

```js
db.events.createIndex({ "$**": 1 })
db.events.createIndex({ "metadata.$**": 1 })
db.events.createIndex({ "$**": 1 }, { wildcardProjection: { important: 0 } })
```

→ 모든 필드를 자동 인덱스. 비용 ↑, 신중히.

---

## 9. TTL Index — 자동 만료

```js
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 })

// 또는 절대 시간 대신 N 초 후
db.logs.createIndex({ createdAt: 1 }, { expireAfterSeconds: 86400 })
// → createdAt 으로부터 24시간 후 삭제
```

- Date 필드만
- 백그라운드 task 가 60 초마다 검사 → 정확 시각 X
- 자세히 → 캐시 / 세션 / 로그 retention 에 표준

---

## 10. Partial Index

조건 만족 행만 인덱스:

```js
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { active: true } }
)
```

→ 인덱스 크기 ↓, 갱신 비용 ↓.
쿼리가 같은 조건 포함해야 사용.

---

## 11. Sparse Index

필드 없는 document 제외:

```js
db.users.createIndex({ phone: 1 }, { sparse: true })
```

`Partial Index` 의 더 일반화된 형태 — Partial 권장.

---

## 12. Unique Index

```js
db.users.createIndex({ email: 1 }, { unique: true })

// 부분 unique
db.users.createIndex(
  { email: 1 },
  { unique: true, partialFilterExpression: { active: true } }
)
```

⚠️ NULL 도 1 개만 유일 — 여러 NULL 필요하면 `sparse: true`.

---

## 13. Index Build

### 13.1 Foreground vs Background

- 4.0 이전 — background option 있었음
- **4.2+** 모든 index build 는 hybrid (사실상 background)
- `commitQuorum` 으로 replica set 전체 commit 조정

### 13.2 진행 상태

```js
db.currentOp({ "command.createIndexes": { $exists: true } })
```

---

## 14. explain 으로 인덱스 검증

```js
db.users.find({ email: "a@x.com" }).explain("executionStats")
```

핵심 필드:
- `winningPlan.stage` — `IXSCAN` / `COLLSCAN` / `FETCH` / `IDHACK`
- `indexName`
- `totalKeysExamined` / `totalDocsExamined` / `nReturned`
- `executionTimeMillis`

### 14.1 신호

| 신호 | 의미 |
| --- | --- |
| `COLLSCAN` | 인덱스 X — 풀스캔 |
| `IXSCAN` only | ✨ Covered Query (FETCH 없음) |
| `totalDocsExamined >> nReturned` | 인덱스 부족 / 비효율 |
| `keysExamined : returned ≈ 1:1` | 좋음 |

---

## 15. Covered Query — Index Only Read

쿼리가 projection 의 모든 필드를 인덱스에서 충당 → 데이터 fetch X.

```js
db.users.createIndex({ email: 1, name: 1 })

db.users.find(
  { email: "a@x.com" },
  { _id: 0, email: 1, name: 1 }    // _id 제외 필수!
).explain("executionStats")
```

조건:
- 모든 query / projection 필드가 인덱스에 있음
- `_id: 0` (인덱스에 _id 없으면)
- 배열 필드 X

---

## 16. 인덱스 진단

```js
// 사용 통계 (3.2+)
db.users.aggregate([{ $indexStats: {} }])

// 크기
db.users.stats().indexSizes

// 안 쓰는 인덱스 찾기
db.users.aggregate([
  { $indexStats: {} },
  { $match: { "accesses.ops": 0 } }
])

// 숨기기 (4.4+)
db.users.hideIndex("idx_old")
db.users.unhideIndex("idx_old")
```

---

## 17. 함정

### 함정 1 — 너무 많은 인덱스
write 마다 모두 갱신. 정기적으로 사용 통계 확인.

### 함정 2 — `_id` 인덱스 의존
모든 컬렉션 _id 자동 unique. 다른 unique 키도 인덱스.

### 함정 3 — 배열 안 객체 인덱스
`{ "arr.field": 1 }` — 매칭 의미 주의 (`$elemMatch` 다름).

### 함정 4 — Compound 인덱스 컬럼 순서
ESR (Equality, Sort, Range).

### 함정 5 — 정규식 인덱스
`/^prefix/` 만 인덱스. 와일드카드 시작은 풀스캔.

### 함정 6 — `count()` 가 인덱스 활용 못 함
`countDocuments` 도 조건 있으면 인덱스 검사. 빈 조건은 `estimatedDocumentCount` 가 빠름.

### 함정 7 — Hidden 인덱스 확인 안 하고 DROP
hide → 모니터링 → drop.

### 함정 8 — TTL 인덱스의 정확 시각 기대
60초 단위 검사. ±60초 오차.

---

## 18. 학습 자료

- **MongoDB Indexes** — docs.mongodb.com/manual/indexes
- **MongoDB University** — M201 (Performance)
- **The ESR Rule** — MongoDB blog

---

## 19. 관련

- [[crud-syntax]] — explain
- [[data-modeling]]
- [[mongodb]] — MongoDB hub
