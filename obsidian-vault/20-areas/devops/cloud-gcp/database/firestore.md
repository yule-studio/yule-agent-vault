---
title: "GCP Firestore"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:50:00+09:00
tags:
  - gcp
  - database
  - firestore
  - nosql
---

# GCP Firestore

**[[database|↑ Database]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 한 줄

매니지드 **Document NoSQL** — 실시간 listener + 모바일 / web SDK + serverless.

옛 Firebase Cloud Firestore = 같은 서비스. Datastore 의 후계.

---

## 2. 두 모드

| | Native | Datastore |
| --- | --- | --- |
| 권장 | ✅ 신규 | 옛 호환 |
| 실시간 | ✅ | X |
| 모바일 SDK | ✅ | X |

→ 신규 = Native.

---

## 3. 데이터 모델

```
Collection (users)
  └── Document (user-123)
        ├── name: "Alice"
        ├── email: "a@x.com"
        └── Subcollection (orders)
              └── Document (order-1)
```

문서 ≤ 1 MB. 중첩 가능. transaction / batch 지원.

---

## 4. 설치 / 사용

```bash
gcloud firestore databases create --location=asia-northeast3
```

```python
from google.cloud import firestore
db = firestore.Client()

# write
db.collection("users").document("alice").set({
    "email": "a@x.com",
    "age": 30
})

# read
doc = db.collection("users").document("alice").get()
print(doc.to_dict())

# query
results = db.collection("users").where("age", ">", 18).limit(10).stream()
for r in results:
    print(r.id, r.to_dict())

# real-time listen
def on_snapshot(docs, changes, time):
    for change in changes:
        print(change.type.name, change.document.id)
db.collection("users").on_snapshot(on_snapshot)
```

### 4.1 모바일 SDK

```javascript
import { initializeApp } from 'firebase/app';
import { getFirestore, doc, onSnapshot } from 'firebase/firestore';

const db = getFirestore(initializeApp(config));
onSnapshot(doc(db, "users", "alice"), (snap) => {
  console.log(snap.data());
});
```

→ 직접 클라이언트 ↔ Firestore. 보안 = Security Rules.

---

## 5. Security Rules

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read: if request.auth.uid == userId;
      allow write: if request.auth.uid == userId
                   && request.resource.data.email is string;
    }
  }
}
```

→ 서버 없이 클라이언트 직접 access 안전.

---

## 6. 비용

```
Read:    $0.06 / 100K
Write:   $0.18 / 100K
Delete:  $0.02 / 100K
Storage: $0.18 / GB·월
Egress:  region 내부 무료

Free tier: 50K read / 20K write / 1 GiB
```

→ 작은 / 중간 = 무료. 거대 read = 폭증 — cache 신중.

---

## 7. 사용 시나리오

- 모바일 / web 앱 backend
- 실시간 (chat, presence)
- user profile / preferences
- 작은 서비스 (auth 통합)

부적합:
- 복잡 query / JOIN
- ad-hoc analytics (BigQuery)
- 큰 row (1 MB+)
- 매우 거대 (Bigtable / Spanner)

---

## 8. 함정

- query 가 인덱스 기반 — 새 query = 새 composite index
- write 의 cost — 매 field 갱신 시 count
- listener 의 over-read
- 1 document 의 1 sec/write 한계 (단일 row hot spot)

---

## 9. 관련

- [[database]]
- [[../../../database/mongodb/mongodb]] — 비교 Document
- [[../security/iam]]
