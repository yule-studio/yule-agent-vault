---
title: "MongoDB Aggregation Pipeline"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:50:00+09:00
tags:
  - database
  - mongodb
  - aggregation
---

# MongoDB Aggregation Pipeline

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 단계 / 연산자 / 패턴 |

**[[mongodb|↑ MongoDB hub]]**

---

## 1. 한 줄

Aggregation Pipeline = **단계의 시퀀스** — 각 단계가 document 스트림을 변환.
SQL 의 `SELECT ... GROUP BY ... HAVING ... JOIN` 보다 표현력 강함.

---

## 2. 기본

```js
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$userId", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } },
  { $limit: 10 }
])
```

| 단계 | 의미 |
| --- | --- |
| `$match` | 필터 (WHERE) |
| `$group` | 집계 (GROUP BY) |
| `$sort` | 정렬 |
| `$limit` | LIMIT |

---

## 3. 주요 단계

### 3.1 $match
WHERE 절. **인덱스 사용** — 파이프라인 앞쪽에 둘 것.

```js
{ $match: { status: "completed", amount: { $gte: 100 } } }
```

### 3.2 $project
컬럼 선택 / 계산.

```js
{ $project: {
    _id: 0,
    email: 1,
    domain: { $arrayElemAt: [{ $split: ["$email", "@"] }, 1] },
    isAdult: { $gte: ["$age", 18] }
  }
}
```

### 3.3 $addFields / $set
필드 추가 (다른 필드 유지).

```js
{ $addFields: { fullName: { $concat: ["$first", " ", "$last"] } } }
```

### 3.4 $group
집계.

```js
{ $group: {
    _id: "$userId",                     // GROUP BY
    total: { $sum: "$amount" },
    avg:   { $avg: "$amount" },
    min:   { $min: "$amount" },
    max:   { $max: "$amount" },
    count: { $sum: 1 },
    items: { $push: "$item" },           // 배열로 모음
    distinctItems: { $addToSet: "$item" }
  }
}

// 전체
{ $group: { _id: null, total: { $sum: "$amount" } } }
```

### 3.5 $sort / $skip / $limit

```js
{ $sort: { createdAt: -1, _id: 1 } }
{ $skip: 100 }                           // ⚠️ 큰 값 느림
{ $limit: 10 }
```

### 3.6 $unwind — 배열 → 행

```js
// { _id:1, tags:["a","b"] } → { _id:1, tags:"a" }, { _id:1, tags:"b" }

{ $unwind: "$tags" }
{ $unwind: { path: "$tags", preserveNullAndEmptyArrays: true, includeArrayIndex: "tagIdx" } }
```

### 3.7 $lookup — JOIN

```js
{ $lookup: {
    from: "orders",
    localField: "_id",
    foreignField: "userId",
    as: "orders"
  }
}

// pipeline 형식 (더 유연)
{ $lookup: {
    from: "orders",
    let: { uid: "$_id" },
    pipeline: [
      { $match: { $expr: { $eq: ["$userId", "$$uid"] } } },
      { $match: { status: "completed" } },
      { $project: { _id: 0, amount: 1 } }
    ],
    as: "orders"
  }
}
```

⚠️ `$lookup` 은 비싸다. 대량은 임베드 / pre-aggregate.

### 3.8 $facet — 멀티 파이프라인

```js
{ $facet: {
    byStatus: [
      { $group: { _id: "$status", count: { $sum: 1 } } }
    ],
    byMonth: [
      { $group: { _id: { $month: "$createdAt" }, total: { $sum: "$amount" } } }
    ],
    top10: [
      { $sort: { amount: -1 } },
      { $limit: 10 }
    ]
  }
}
```

→ 한 번에 여러 분석.

### 3.9 $bucket / $bucketAuto

```js
{ $bucket: {
    groupBy: "$age",
    boundaries: [0, 18, 30, 50, 80],
    default: "Other",
    output: { count: { $sum: 1 } }
  }
}

{ $bucketAuto: { groupBy: "$amount", buckets: 4 } }
```

### 3.10 $count

```js
{ $count: "total" }
```

### 3.11 $merge / $out — 결과 저장

```js
{ $out: "daily_summary" }                  // 컬렉션 통째로 교체

{ $merge: {
    into: "daily_summary",
    on: ["date", "metric"],
    whenMatched: "merge",
    whenNotMatched: "insert"
  }
}
```

→ Materialized view 패턴.

### 3.12 $graphLookup — 트리 / 그래프

```js
{ $graphLookup: {
    from: "categories",
    startWith: "$parentId",
    connectFromField: "parentId",
    connectToField: "_id",
    as: "ancestors",
    maxDepth: 5
  }
}
```

### 3.13 $changeStream
실시간 변경 스트림 (별도 메커니즘이지만 aggregate 위에 구축).

### 3.14 $vectorSearch (Atlas 8.0+)

```js
{ $vectorSearch: {
    index: "vec_idx",
    path: "embedding",
    queryVector: [0.1, 0.2, ...],
    numCandidates: 100,
    limit: 10
  }
}
```

---

## 4. Expression 연산자

### 4.1 산술

```js
$add, $subtract, $multiply, $divide, $mod
$pow, $sqrt, $abs, $exp, $log, $log10, $ln
$ceil, $floor, $round, $trunc
```

### 4.2 비교

```js
$eq, $ne, $gt, $gte, $lt, $lte
$cmp
```

### 4.3 논리

```js
$and, $or, $not, $nor
$cond: [<if>, <then>, <else>]
$ifNull: ["$field", "default"]
$switch: { branches: [...], default: ... }
```

### 4.4 문자열

```js
$concat, $substr, $toLower, $toUpper
$split, $strLenCP, $indexOfCP
$trim, $ltrim, $rtrim
$regexMatch, $regexFind, $regexFindAll
$dateToString
```

### 4.5 배열

```js
$size, $arrayElemAt, $first, $last, $slice
$concatArrays, $reverseArray
$filter: { input, cond, as }
$map: { input, as, in }
$reduce: { input, initialValue, in }
$zip
$in: ["x", "$arr"]
$arrayToObject, $objectToArray
```

### 4.6 날짜

```js
$year, $month, $dayOfMonth, $hour, $minute, $second
$week, $isoWeek, $dayOfYear, $dayOfWeek
$dateAdd, $dateSubtract, $dateDiff
$dateFromString, $dateToString
$dateTrunc, $dateToParts
```

### 4.7 조건

```js
{ $cond: { if: { $gte: ["$age", 18] }, then: "adult", else: "minor" } }

{ $switch: {
    branches: [
      { case: { $lt: ["$age", 18] }, then: "minor" },
      { case: { $lt: ["$age", 65] }, then: "adult" }
    ],
    default: "senior"
  }
}
```

---

## 5. $expr — find 안에서 expression

```js
db.users.find({
  $expr: { $gt: ["$balance", "$creditLimit"] }
})
```

같은 document 안의 필드 비교.

---

## 6. Index 사용

`$match` 와 `$sort` 가 파이프라인 **앞** 에 있으면 인덱스 사용.

```js
// ✅ 인덱스 사용
[
  { $match: { userId: 1 } },          // 인덱스
  { $sort: { createdAt: -1 } },
  { $group: ... }
]

// ❌ 인덱스 사용 X
[
  { $group: ... },
  { $match: { count: { $gt: 10 } } }   // 그룹 후의 필드
]
```

---

## 7. explain

```js
db.orders.explain("executionStats").aggregate([...])
```

각 stage 의 `$cursor`, `inputStage`, `executionTimeMillisEstimate` 확인.

---

## 8. 윈도우 함수 ($setWindowFields) — 5.0+

```js
db.sales.aggregate([
  { $setWindowFields: {
      partitionBy: "$userId",
      sortBy: { createdAt: 1 },
      output: {
        rolling7Sum: {
          $sum: "$amount",
          window: { documents: [-6, 0] }      // 최근 7
        },
        rank: { $rank: {} },
        runningTotal: {
          $sum: "$amount",
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  }
])
```

SQL Window 와 거의 동일.

---

## 9. 자주 쓰는 패턴

### 9.1 페이지네이션 + 카운트

```js
db.users.aggregate([
  { $match: { active: true } },
  { $facet: {
      data: [
        { $sort: { createdAt: -1 } },
        { $skip: 20 }, { $limit: 10 }
      ],
      total: [
        { $count: "count" }
      ]
    }
  }
])
```

### 9.2 Top-N per group

```js
db.posts.aggregate([
  { $sort: { createdAt: -1 } },
  { $group: {
      _id: "$userId",
      latest: { $push: "$$ROOT" }
    }
  },
  { $project: {
      latest: { $slice: ["$latest", 3] }
    }
  }
])
```

### 9.3 시계열 집계

```js
db.metrics.aggregate([
  { $match: { ts: { $gte: ISODate("2026-05-01") } } },
  { $group: {
      _id: {
        $dateTrunc: { date: "$ts", unit: "hour" }
      },
      avg: { $avg: "$value" }
    }
  },
  { $sort: { _id: 1 } }
])
```

### 9.4 두 컬렉션 JOIN + 집계

```js
db.users.aggregate([
  { $lookup: {
      from: "orders",
      localField: "_id",
      foreignField: "userId",
      as: "orders"
    }
  },
  { $project: {
      name: 1,
      orderCount: { $size: "$orders" },
      totalSpent: { $sum: "$orders.amount" }
    }
  }
])
```

---

## 10. 함정

### 10.1 `$lookup` 의 대용량
N × M 비용. 가능하면 임베드 / pre-aggregate / $merge.

### 10.2 `$unwind` 의 폭발
배열 element 만큼 행 늘어남.

### 10.3 메모리 한계
파이프라인 stage 100 MB 한계. 넘으면 `allowDiskUse: true`.

```js
db.x.aggregate([...], { allowDiskUse: true })
```

### 10.4 `$match` 위치
앞에 둬야 인덱스 사용 + 데이터 줄어듦.

### 10.5 `$skip` 큰 값
페이지네이션 안티패턴. cursor 기반.

### 10.6 `$group` 결과 16MB
한 그룹의 `$push` 가 폭증. 임시 `$out` 또는 다른 패턴.

### 10.7 `$facet` 비용
각 sub-pipeline 모두 실행. 가벼울 때만.

---

## 11. 학습 자료

- **MongoDB Aggregation** — docs.mongodb.com/manual/aggregation
- **MongoDB University** — M121 (Aggregation Framework)
- **Practical MongoDB Aggregations** — Paul Done (무료)

---

## 12. 관련

- [[crud-syntax]] — 기본 쿼리
- [[indexes]] — $match / $sort 인덱스
- [[mongodb]] — MongoDB hub
