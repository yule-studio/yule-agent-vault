---
title: "MongoDB 시작하기 — 설치 / mongosh / 첫 명령"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:25:00+09:00
tags:
  - database
  - mongodb
  - setup
---

# MongoDB 시작하기

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 설치 / 첫 명령 |

**[[mongodb|↑ MongoDB hub]]**

---

## 1. 설치

### 1.1 macOS

```bash
brew tap mongodb/brew
brew install mongodb-community@7.0
brew services start mongodb-community@7.0

brew install mongosh
```

### 1.2 Ubuntu

```bash
# 공식 repo
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-7.0.gpg
echo "deb [signed-by=...] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt update
sudo apt install -y mongodb-org

sudo systemctl enable --now mongod
```

### 1.3 Docker

```bash
docker run -d \
  --name mongo7 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=secret \
  -p 27017:27017 \
  -v mongo7-data:/data/db \
  mongo:7.0
```

### 1.4 매니지드

| 서비스 | 특징 |
| --- | --- |
| **MongoDB Atlas** | 공식 매니지드 (AWS/GCP/Azure) |
| Amazon DocumentDB | API 호환 (일부) |
| Azure Cosmos DB (Mongo API) | API 호환 |

Atlas 가 사실상 표준 — vector search / Atlas Search 등 풍부.

---

## 2. mongosh — Shell

```bash
mongosh                              # localhost
mongosh "mongodb://localhost:27017"
mongosh "mongodb://user:pass@host/db?authSource=admin"
mongosh "mongodb+srv://cluster.xxx.mongodb.net/db" --username user   # Atlas
```

---

## 3. 첫 명령

```js
// 데이터베이스 / collection
show dbs
use app                              // 없으면 생성
show collections
db.users.find()

// 데이터 삽입
db.users.insertOne({
  name: "Alice",
  email: "alice@example.com",
  age: 30,
  tags: ["admin", "active"],
  createdAt: new Date()
})

db.users.insertMany([
  { name: "Bob",   age: 25 },
  { name: "Carol", age: 35 }
])

// 조회
db.users.find({ age: { $gte: 30 } })
db.users.findOne({ email: "alice@example.com" })
db.users.find().pretty()
db.users.find().sort({ age: -1 }).limit(10)

// 수정
db.users.updateOne(
  { email: "alice@example.com" },
  { $set: { age: 31 }, $push: { tags: "vip" } }
)

// 삭제
db.users.deleteOne({ email: "bob@example.com" })
db.users.deleteMany({ age: { $lt: 18 } })

// 카운트 / 통계
db.users.countDocuments({ age: { $gte: 18 } })
db.users.estimatedDocumentCount()
db.users.stats()
db.stats()
```

---

## 4. ObjectId & _id

```js
db.users.insertOne({ name: "Eve" })
// 자동 _id 생성:
// ObjectId("6648a3f4...") — 12 byte
//   timestamp(4) + random(5) + counter(3)
```

직접 지정:
```js
db.users.insertOne({ _id: "alice", name: "Alice" })
```

UUID / 문자열도 가능. ObjectId 가 표준.

---

## 5. 인증 / 권한

### 5.1 사용자 생성

```js
use admin
db.createUser({
  user: "admin",
  pwd: "secret",
  roles: ["root"]
})

use app
db.createUser({
  user: "app_rw",
  pwd: "secret",
  roles: [
    { role: "readWrite", db: "app" }
  ]
})

db.createUser({
  user: "app_ro",
  pwd: "secret",
  roles: [
    { role: "read", db: "app" }
  ]
})
```

### 5.2 인증 활성화

```yaml
# mongod.conf
security:
  authorization: enabled
```

이후:
```bash
mongosh "mongodb://app_rw:secret@localhost/app?authSource=app"
```

### 5.3 역할

| Role | 의미 |
| --- | --- |
| `read` | 읽기 |
| `readWrite` | CRUD |
| `dbAdmin` | 인덱스 / 통계 / 검증 |
| `userAdmin` | 사용자 관리 |
| `clusterAdmin` | 클러스터 |
| `root` | 모두 |

---

## 6. 연결 문자열

```
mongodb://[user:pass@]host[:27017]/db[?option=value&...]
mongodb+srv://user:pass@cluster.xxx.mongodb.net/db   # Atlas (SRV)

옵션:
  authSource=admin
  replicaSet=rs0
  ssl=true
  retryWrites=true
  w=majority
  readPreference=secondaryPreferred
```

---

## 7. 자주 쓰는 shell 명령

```js
db.version()
db.serverStatus()
db.runCommand({ ping: 1 })
db.adminCommand({ getParameter: "*" })
db.currentOp()                                // 활성 op
db.killOp(<opid>)

db.users.getIndexes()
db.users.createIndex({ email: 1 }, { unique: true })
db.users.dropIndex("email_1")

db.users.explain("executionStats").find({ age: 30 })
```

---

## 8. 백업 / 복원

```bash
# 단일 DB
mongodump --uri "mongodb://localhost/app" --out /backup
mongorestore --uri "mongodb://localhost/app" /backup/app

# 단일 collection
mongoexport --uri "..." --collection users --out users.json
mongoimport --uri "..." --collection users --file users.json

# Atlas — 매니지드 자동
```

자세히 → [[replication]]

---

## 9. compass — GUI

[mongodb.com/compass](https://www.mongodb.com/products/compass) — 공식 GUI.
- 시각적 쿼리 빌더
- 인덱스 / 스키마 분석
- Aggregation Pipeline editor

---

## 10. 함정

### 함정 1 — 인증 없이 외부 노출
**누구나 접근**. 흔한 침해 사고. 항상 `authorization: enabled` + 강한 비번.

### 함정 2 — 단일 노드 운영
Replica Set 아니면 트랜잭션 X, failover X. 신규는 항상 RS.

### 함정 3 — 모든 권한 `root`
권한 분리.

### 함정 4 — Schema 없다고 안 만들기
검증 / 인덱스 / 일관성 — schema validator + 어플리케이션 모델 필수.

### 함정 5 — 큰 document
16 MB 한계. 가까워지면 분할.

### 함정 6 — `_id` 의 무작위 UUIDv4
인덱스 fragmentation. ObjectId / UUIDv7.

### 함정 7 — 27017 외부 노출
방화벽 / Atlas IP allowlist.

---

## 11. 관련

- [[configuration]] — mongod.conf
- [[crud-syntax]] — CRUD
- [[mongodb]] — MongoDB hub
