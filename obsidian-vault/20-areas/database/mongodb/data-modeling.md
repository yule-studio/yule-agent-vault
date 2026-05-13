---
title: "MongoDB 데이터 모델링 — 임베드 vs 참조"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:35:00+09:00
tags:
  - database
  - mongodb
  - modeling
---

# MongoDB 데이터 모델링

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 임베드 / 참조 / 패턴 |

**[[mongodb|↑ MongoDB hub]]**

---

## 1. Document 모델 한 줄

> "**같이 쓰는 것은 같이 저장**" — Relational 의 정규화와 반대 방향.

RDB 처럼 모든 것을 분해하면 안 됨 — JOIN 비용이 큰 MongoDB 에선 안티패턴.

---

## 2. BSON Document

```json
{
  "_id": ObjectId("..."),
  "name": "Alice",
  "email": "alice@x.com",
  "age": 30,
  "tags": ["admin", "active"],
  "addresses": [
    { "type": "home", "city": "Seoul", "zip": "12345" },
    { "type": "work", "city": "Seoul", "zip": "67890" }
  ],
  "settings": { "theme": "dark", "notifications": true },
  "createdAt": ISODate("2026-05-13T...")
}
```

특징:
- 한 document 최대 **16 MB**
- 중첩 / 배열 자유
- 타입 다양 (String, Number, Date, Boolean, ObjectId, Decimal128, Binary, ...)

---

## 3. 임베드 (Embed) — One-to-Few / One-to-Many

### 3.1 예 — 사용자의 주소

```js
// Embed
{
  _id: 1,
  name: "Alice",
  addresses: [
    { city: "Seoul", zip: "12345" },
    { city: "Busan", zip: "67890" }
  ]
}
```

### 3.2 장점
- **1 query 로 모두** 가져옴
- 일관성 (한 document 의 update 는 원자적)
- 빠름

### 3.3 단점
- Document 크기 증가
- 같은 데이터 중복 (denormalize)
- 변경 시 여러 곳 갱신 (만약 공유 데이터면)

### 3.4 언제
- One-to-Few (수 십 개 이하)
- 함께 조회되는 데이터
- 변경 빈도 낮음

---

## 4. 참조 (Reference) — One-to-Many / Many-to-Many

### 4.1 예 — 사용자 ↔ 주문 (수천 개)

```js
// users
{ _id: 1, name: "Alice" }

// orders
{ _id: 100, userId: 1, total: 50 }
{ _id: 101, userId: 1, total: 30 }
```

### 4.2 조회 — $lookup

```js
db.users.aggregate([
  { $match: { _id: 1 } },
  { $lookup: {
      from: "orders",
      localField: "_id",
      foreignField: "userId",
      as: "orders"
    }
  }
])
```

### 4.3 장점
- Document 크기 일정
- 공유 데이터 1 곳에
- 큰 컬렉션 가능

### 4.4 단점
- 2 query (또는 $lookup) — JOIN 비용
- 트랜잭션 필요할 수도

### 4.5 언제
- One-to-Many (수백+ 이상)
- 자식이 독립적으로 변경
- 공유 데이터

---

## 5. 결정 가이드 — Rule of Thumb

| 관계 | 카디널리티 | 권장 |
| --- | --- | --- |
| One-to-Few | < 수십 | 임베드 |
| One-to-Many | 수백~수천 | 참조 |
| One-to-Squillions | 수백만 | 부모 → 자식의 참조 |
| Many-to-Many | 양쪽 큼 | 양방향 참조 (`ids: []`) |
| 변경 빈도 ↑ | 어디든 | 참조 (denormalize 회피) |
| 함께 읽음 | 어디든 | 임베드 |

---

## 6. 자주 쓰는 모델링 패턴

### 6.1 Embed 패턴

```js
// 사용자 + 작은 프로필
{
  _id: 1,
  email: "...",
  profile: { name: "Alice", bio: "..." }
}
```

### 6.2 Reference 패턴

```js
// 사용자 + 거대 게시글
{ _id: 1, name: "Alice" }
{ _id: 100, authorId: 1, title: "...", body: "..." }
```

### 6.3 Bucket Pattern (시계열)

```js
// 분당 버킷
{
  device: "sensor-1",
  bucket: ISODate("2026-05-13T10:00:00Z"),
  count: 60,
  values: [
    { t: ISODate("...10:00:01Z"), v: 21.5 },
    { t: ISODate("...10:00:02Z"), v: 21.6 },
    ...
  ]
}
```

→ 분당 60 doc 대신 1 doc. 인덱스 / 메모리 효율 ↑.
MongoDB 5.0+ 의 **Time Series Collection** 이 자동화.

### 6.4 Schema Versioning Pattern

```js
{
  _id: 1,
  schemaVersion: 2,
  ...
}
```

마이그레이션 부담 ↓ — 응용에서 분기.

### 6.5 Computed Pattern

```js
// 매번 계산하지 않고 저장
{ _id: 1, orderCount: 42, totalSpent: 1234.5 }
```

집계가 비쌀 때.

### 6.6 Subset Pattern

```js
// 자주 쓰는 부분만 임베드, 나머지는 참조
{
  _id: 1,
  name: "Alice",
  recentOrders: [    // 최근 10 개만 임베드
    { id: 100, total: 50 }, ...
  ]
}
// 전체는 orders 컬렉션
```

### 6.7 Outlier Pattern

대부분 작은 컬렉션이지만 일부 거대 → 거대는 참조로:

```js
{ _id: 1, comments: [...], hasMoreComments: true }
// 별도 comments 컬렉션
```

### 6.8 Tree / Hierarchy

```js
// Parent Reference
{ _id: "/A/B", parent: "/A", name: "B" }

// Array of Ancestors
{ _id: "B", ancestors: ["root", "A"], name: "B" }

// Materialized Path
{ _id: "B", path: ",root,A,", name: "B" }
```

`Array of Ancestors` 가 가장 query 친화적.

### 6.9 Polymorphic Pattern

같은 컬렉션에 type 별 다른 schema:
```js
{ _id: 1, type: "book", title: "...", author: "..." }
{ _id: 2, type: "movie", title: "...", director: "..." }
```

`type` 인덱스 + 응용 분기.

---

## 7. _id 전략

| 전략 | 예 | 특징 |
| --- | --- | --- |
| ObjectId (기본) | `ObjectId("...")` | 시간 정렬 + 작음 |
| UUID v4 | `UUID("...")` | 무작위, 인덱스 ↓ |
| UUID v7 / ULID | 시간 정렬 UUID | 권장 |
| 자체 의미 | `"user:alice"` | 자연 키 |

ObjectId 가 표준. 외부 시스템 호환 등으로 UUID 필요하면 v7.

---

## 8. 인덱스와 모델링

같이 가야 함. 자세히 → [[indexes]]

```js
// 자주 조회되는 필드 인덱스
db.users.createIndex({ email: 1 }, { unique: true })
db.orders.createIndex({ userId: 1, createdAt: -1 })

// 배열 필드 — multi-key index
db.users.createIndex({ tags: 1 })

// 중첩
db.users.createIndex({ "addresses.city": 1 })
```

---

## 9. 트랜잭션 vs 모델링

4.0+ 부터 multi-document ACID. 하지만 **임베드로 표현 가능하면 임베드** 가 더 빠름.

```js
// ❌ 두 컬렉션에 트랜잭션
session.startTransaction()
db.accounts.updateOne({_id:1}, {$inc:{balance:-100}}, {session})
db.accounts.updateOne({_id:2}, {$inc:{balance:+100}}, {session})
session.commitTransaction()

// ✅ 단일 document (가능하면)
// 사용자 안에 모든 잔액
```

---

## 10. Schema Validation

```js
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email"],
      properties: {
        email: { bsonType: "string" },
        age:   { bsonType: "int", minimum: 0 }
      }
    }
  }
})
```

스키마 자유 + 검증 = 좋은 조합.

---

## 11. 안티패턴

### 11.1 Massive Arrays
한 document 안에 수만 element 배열. 16 MB 한계 + 인덱스 비효율.

### 11.2 Unbounded Document
계속 자라는 document. Bucket / Subset 패턴.

### 11.3 Over-normalized
RDB 처럼 모든 것을 분해. $lookup 폭증.

### 11.4 Over-embedded
공유 데이터를 임베드. 변경 시 여러 곳 갱신.

### 11.5 Document 안에 큰 binary
이미지 / 파일 — S3 + URL 만.

---

## 12. 회사 사례

| 회사 | MongoDB 사용 |
| --- | --- |
| Adobe | 콘텐츠 메타 |
| eBay | 일부 |
| Forbes | CMS |
| Sega | 게임 데이터 |
| Trello | 카드 / 보드 |
| Uber | 일부 (특정 도메인) |

대부분 도메인 모델이 자연스럽게 document 인 경우.

---

## 13. 함정

### 함정 1 — RDB 사고로 설계
모두 분해 → 비효율. document 사고로.

### 함정 2 — Document 16 MB 한계
점차 커지는 document → 분할 패턴.

### 함정 3 — 배열 무한 증가
인덱스 / 메모리 폭발.

### 함정 4 — Schema 없으니 일관성 X
validator + 어플리케이션 모델 필수.

### 함정 5 — $lookup 남용
대량 → 느림. 임베드 / pre-computed.

### 함정 6 — 무작위 _id (UUIDv4)
인덱스 fragmentation. ObjectId / UUIDv7.

---

## 14. 학습 자료

- **MongoDB Data Modeling** — docs.mongodb.com/manual/data-modeling
- **Building with Patterns** — 12-part series (mongodb.com/blog)
- **MongoDB University** — M320: Data Modeling

---

## 15. 관련

- [[crud-syntax]] — CRUD
- [[indexes]] — 인덱스
- [[aggregation]] — $lookup
- [[mongodb]] — MongoDB hub
