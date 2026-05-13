---
title: "MongoDB 보안 — --auth / role / 분실 복구"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:30:00+09:00
tags:
  - database
  - mongodb
  - security
  - password
---

# MongoDB 보안 — --auth / role / 분실 복구

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | --auth / role / 분실 복구 / 외부 노출 |

**[[mongodb|↑ MongoDB hub]]**

> MongoDB 도 Redis 와 함께 **인증 없이 인터넷에 열려 데이터 삭제 / 랜섬** 사고가 가장 많은 DB.
> 일반 파라미터는 [[configuration]].

---

## 1. 왜 MongoDB 보안이 특히 위험한가

2017년 "MongoDB ransomware" 사고 — 수만 개 인증 없는 MongoDB 가 인터넷에 열려 있었고, 봇이 데이터 dump → drop → 비트코인 요구.

기본 동작:
- 3.6 이전: `bindIp` 가 모든 인터페이스
- `--auth` 안 켜면 누구나 admin
- shodan / censys 로 항상 스캔 중

→ 어떤 환경이든 **`--auth` + `bindIp` 제한** 이 최소선.

---

## 2. 인증 켜기 — 가장 먼저 할 일

### 2.1 새 설치 — localhost exception 으로 admin 만들기

3.6+ 는 `bindIp: 127.0.0.1` + `--auth` 미설정 상태에서 **localhost 에서 첫 admin 생성** 까지만 인증 없이 허용 (localhost exception).

```yaml
# /etc/mongod.conf
security:
  authorization: enabled
net:
  bindIp: 127.0.0.1
  port: 27017
```

```bash
# 1. auth 안 켠 상태로 띄움 (또는 켠 상태 + localhost)
mongosh

# 2. admin 사용자 생성
use admin
db.createUser({
  user: "admin",
  pwd: passwordPrompt(),                  // 평문 입력 방지
  roles: [ { role: "userAdminAnyDatabase", db: "admin" },
           { role: "readWriteAnyDatabase", db: "admin" } ]
})

// 3. auth 켜고 재기동
exit
```

```bash
sudo systemctl restart mongod
mongosh -u admin -p --authenticationDatabase admin
```

### 2.2 앱 사용자 — 최소 권한

```javascript
use app
db.createUser({
  user: "app_rw",
  pwd: passwordPrompt(),
  roles: [ { role: "readWrite", db: "app" } ]
})

db.createUser({
  user: "app_ro",
  pwd: passwordPrompt(),
  roles: [ { role: "read", db: "app" } ]
})
```

내장 role 일부:

| Role | 의미 |
| --- | --- |
| `read` / `readWrite` | DB 단위 |
| `dbAdmin` | 인덱스 / stats / drop collection |
| `userAdmin` | 그 DB 의 사용자 관리 |
| `clusterAdmin` | replica set / sharding |
| `readAnyDatabase` / `readWriteAnyDatabase` | 모든 DB (admin DB 부여) |
| `root` | 모든 권한 |

**커스텀 role**:

```javascript
db.createRole({
  role: "appBackup",
  privileges: [
    { resource: { db: "app", collection: "" }, actions: ["find"] }
  ],
  roles: []
})
```

### 2.3 패스워드 변경

```javascript
use admin
db.changeUserPassword("app_rw", passwordPrompt())

// 또는
db.updateUser("app_rw", { pwd: passwordPrompt() })
```

### 2.4 패스워드 평문 "찾기" 는 불가

MongoDB 도 SCRAM-SHA-256 해시 저장 (`admin.system.users` 의 `credentials`). 복호화 불가.

```javascript
use admin
db.system.users.find({ user: "app_rw" })
// credentials: { 'SCRAM-SHA-256': { iterationCount, salt, storedKey, serverKey } }
```

분실하면 **재설정**.

---

## 3. 패스워드를 잊어버렸을 때

### 3.1 admin 패스워드 분실

> **전제: 서버 호스트 OS 접근.**

```bash
# 1. auth 끄고 재기동 + localhost 만
sudo systemctl stop mongod

# /etc/mongod.conf — security.authorization 주석 처리
sudo vi /etc/mongod.conf
# security:
#   authorization: enabled     ← 주석

# bindIp 도 임시로 127.0.0.1 만 (외부 차단)
# net.bindIp: 127.0.0.1

sudo systemctl start mongod

# 2. 패스워드 변경
mongosh
use admin
db.changeUserPassword("admin", passwordPrompt())

# 3. auth 다시 켜고 재기동
# /etc/mongod.conf 원복
sudo systemctl restart mongod
```

**중요**: auth 끈 사이 외부 노출되면 안 됨. `bindIp: 127.0.0.1` 반드시 같이.

### 3.2 일반 사용자 분실

admin 으로 들어가서 변경.

```javascript
use admin
db.changeUserPassword("app_rw", passwordPrompt())
```

---

## 4. 외부 노출 — 사고 패턴

### 4.1 `bindIp` 함정

3.6 이전: 기본 `0.0.0.0` → 인터넷 열림.
3.6+: 기본 `127.0.0.1`.

```yaml
# /etc/mongod.conf
net:
  bindIp: 127.0.0.1,10.0.1.5     # localhost + 내부 인터페이스만
  # bindIpAll: true              # ⚠️ 절대 X
  port: 27017
```

### 4.2 사고 시나리오

```
1. MongoDB 띄움 — bindIp: 0.0.0.0, --auth 미설정
2. 방화벽 없음
3. Shodan 등록 후 수 시간 내 봇 접속
4. db.dropDatabase() / db.collection.drop()
5. README 컬렉션에 "0.05 BTC 보내면 복구" 메시지
```

방어:
1. `--auth` 무조건 켜기
2. `bindIp` 127.0.0.1 + 내부 IP 만
3. 방화벽 / SG 로 27017 inbound 제한
4. TLS 강제

### 4.3 Docker 함정

```bash
# ❌ 0.0.0.0 바인드
docker run -p 27017:27017 mongo

# ✅ localhost 만
docker run -p 127.0.0.1:27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD_FILE=/run/secrets/mongo_pwd \
  mongo
```

`MONGO_INITDB_ROOT_PASSWORD` 직접 쓰면 `docker inspect` 에 평문. **secret / env file** 사용.

---

## 5. TLS

```yaml
# /etc/mongod.conf
net:
  tls:
    mode: requireTLS                 # disabled / allowTLS / preferTLS / requireTLS
    certificateKeyFile: /etc/ssl/mongo.pem
    CAFile: /etc/ssl/ca.pem
    allowConnectionsWithoutCertificates: false   # 클라이언트 인증서 강제
```

```bash
mongosh --tls --tlsCAFile ca.pem --tlsCertificateKeyFile client.pem \
  -u admin -p --authenticationDatabase admin
```

x.509 클라이언트 인증으로 패스워드 없는 인증도 가능 (`MONGODB-X509`).

---

## 6. 감사 (audit)

Enterprise 기능. Community 는 `--profile` / log 로 일부.

```yaml
# Enterprise — mongod.conf
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.log
  filter: '{ atype: { $in: ["authenticate", "authCheck", "createUser", "dropUser"] } }'
```

Community 는 mongod 로그 + IDS 로 보완:

```bash
grep -E "authentication failed|SCRAM" /var/log/mongodb/mongod.log
```

---

## 7. Replica Set / Sharding 인증

### 7.1 keyFile (간단)

같은 키 파일을 모든 노드가 공유:

```bash
openssl rand -base64 756 > /etc/mongo/keyfile
chmod 400 /etc/mongo/keyfile
chown mongod:mongod /etc/mongo/keyfile
```

```yaml
# /etc/mongod.conf
security:
  authorization: enabled
  keyFile: /etc/mongo/keyfile

replication:
  replSetName: rs0
```

### 7.2 x.509 (운영 권장)

각 노드가 클러스터 멤버 인증서를 가짐. CN/OU 매칭. keyFile 보다 운영 깔끔.

---

## 8. 운영 보안 체크리스트

- [ ] `--auth` (=`security.authorization: enabled`) 켬
- [ ] admin 사용자 만든 직후 localhost exception 닫힘
- [ ] 앱 사용자 최소 권한 (`readWrite` 만, `clusterAdmin/root` 금지)
- [ ] `bindIp` 는 필요한 인터페이스만 (`bindIpAll` 금지)
- [ ] TLS 강제 (`requireTLS`)
- [ ] 방화벽 / SG 로 27017 inbound 제한
- [ ] Docker 는 `-p 127.0.0.1:27017:27017` 또는 internal network
- [ ] Replica Set 은 `keyFile` 또는 x.509
- [ ] `mongod` OS 유저로 실행 (root 금지)
- [ ] 백업 파일 권한 600 + 암호화 저장
- [ ] 패스워드는 vault 에서 주입 (env / secret manager)

---

## 9. 함정 모음

### 함정 1 — `--auth` 안 켬
설치 직후 가장 흔한 사고. **첫 작업은 admin 만들고 auth 켜기**.

### 함정 2 — admin DB 가 아닌 곳에 admin 만듦
```javascript
// ❌
use app
db.createUser({ user: "admin", ... })       // app DB 의 admin — 의미 없음

// ✅
use admin
db.createUser({ user: "admin", ... })
```

인증할 때 `--authenticationDatabase` 도 admin 으로:
```bash
mongosh -u admin -p --authenticationDatabase admin
```

### 함정 3 — `readWriteAnyDatabase` 를 앱에 부여
앱은 자기 DB 만. anyDatabase 는 admin / 백업 전용.

### 함정 4 — `MONGO_INITDB_ROOT_PASSWORD` 평문 env
`docker inspect` 와 `/proc/<pid>/environ` 에 평문. **`_FILE` suffix 또는 Docker secret**.

### 함정 5 — localhost exception 닫기 전에 외부 노출
첫 admin 만들기 전 `bindIp: 0.0.0.0` 이면 누구나 첫 admin 을 만들 수 있음. **bindIp 좁히기가 먼저**.

### 함정 6 — `passwordPrompt()` 안 쓰고 평문
```javascript
db.createUser({ user: "x", pwd: "plain" })    // 명령 history / log 에 평문
db.createUser({ user: "x", pwd: passwordPrompt() })   // ✅
```

### 함정 7 — keyFile 권한
keyFile 권한이 400 / mongod 소유가 아니면 기동 실패. 클러스터 전체 동일 파일.

### 함정 8 — mongodump 인증
```bash
# 평문 패스워드 노출
mongodump -u admin -p S3cret!

# URI + env 또는 prompt
export MONGODB_URI='mongodb://admin@host/?authSource=admin'
mongodump --uri="$MONGODB_URI"
```

### 함정 9 — 백업 파일 안에 user 정보
`mongodump` 의 `admin` DB 백업에 `system.users` 의 SCRAM 해시 포함. 백업 파일도 권한 / 암호화.

---

## 10. 관련

- [[configuration]] — mongod.conf 일반 파라미터
- [[getting-started]] — 첫 설정
- [[../../security/security|↗ 20-areas/security]]
- [[mongodb]] — MongoDB hub
