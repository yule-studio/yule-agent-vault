---
title: "Redis 보안 — requirepass / ACL / 외부 노출"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:20:00+09:00
tags:
  - database
  - redis
  - security
  - password
---

# Redis 보안 — requirepass / ACL / 외부 노출

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 패스워드 / ACL / 외부 노출 사고 |

**[[redis|↑ Redis hub]]**

> Redis 는 **인증 없는 외부 노출** 이 가장 흔한 사고 패턴.
> 일반 파라미터는 [[configuration]].

---

## 1. 왜 Redis 보안이 특히 위험한가

Redis 는 기본적으로:
- **패스워드 없음** (6 이전 / 잘못된 설정)
- **모든 인터페이스에 바인드** (`bind 0.0.0.0` 인 설정)
- **`CONFIG`, `FLUSHALL`, `DEBUG` 등 위험 명령** 누구나 호출 가능
- **`SLAVEOF` / `MODULE LOAD`** 로 임의 코드 실행 가능

→ 인증 없이 인터넷에 열리면 **수 분 내 RCE / 데이터 파괴**. cryptojacking 봇이 항상 스캔 중.

---

## 2. 패스워드 설정 — 기본

### 2.1 `requirepass` (전통)

```ini
# redis.conf
requirepass S3cret-very-long-string!
```

reload:

```bash
redis-cli -a 'old' CONFIG SET requirepass 'New$ecret'
redis-cli -a 'New$ecret' CONFIG REWRITE   # redis.conf 에 저장
```

접속:

```bash
redis-cli -a 'S3cret-very-long-string!'    # ⚠️ ps 노출
# 또는
redis-cli
127.0.0.1:6379> AUTH S3cret-very-long-string!
```

> **함정**: `-a` 는 `ps` 에 비밀번호 노출. `--no-auth-warning` 추가하거나 `REDISCLI_AUTH` 환경변수 또는 `AUTH` 명령.

```bash
export REDISCLI_AUTH='S3cret!'
redis-cli                                   # 자동 AUTH
```

### 2.2 ACL — 6.0+

`requirepass` 는 default 사용자 (`default`) 의 단일 비번. ACL 은 **사용자별 비번 + 명령/키 제한**.

```bash
# 사용자 추가
ACL SETUSER app_rw on >S3cret! ~app:* +@read +@write +@string +@hash -@dangerous

# 읽기 전용
ACL SETUSER app_ro on >ReadOnly! ~app:* +@read -@write -@dangerous

# default 사용자 잠그기
ACL SETUSER default off                    # ⚠️ 자기 자신 잘리지 않게 조심
# 또는 nopass 만 끄기
ACL SETUSER default on >StrongDefault! ~* +@all
```

ACL 파일 따로:

```ini
# redis.conf
aclfile /etc/redis/users.acl
```

```
# /etc/redis/users.acl
user default off
user app_rw on #<sha256-hex> ~app:* +@read +@write -@dangerous
user admin on >AdminPass! ~* +@all
```

```bash
# 현재 ACL 보기
ACL LIST
ACL WHOAMI
ACL GETUSER app_rw
```

### 2.3 `@category` 와 위험 명령 제외

```
@all          # 전부
@read / @write / @keyspace / @string / @hash / @list / @set / @zset
@admin        # CONFIG, SHUTDOWN, REPLICAOF 등
@dangerous    # FLUSHDB, FLUSHALL, KEYS, DEBUG, MIGRATE, SLAVEOF 등
@scripting    # EVAL, EVALSHA
@pubsub
```

권장 패턴: 앱 사용자는 `+@read +@write -@dangerous -@admin`.

---

## 3. 패스워드를 잊어버렸을 때

### 3.1 `requirepass` 분실

> **전제: 서버 호스트 OS 접근.**

```bash
# 1. Redis 중단
sudo systemctl stop redis

# 2. redis.conf 의 requirepass 라인 주석 처리
sudo vi /etc/redis/redis.conf
# # requirepass old-forgotten-pass

# 3. 재기동
sudo systemctl start redis

# 4. 비밀번호 다시 설정
redis-cli CONFIG SET requirepass 'NewS3cret!'
redis-cli -a 'NewS3cret!' CONFIG REWRITE
```

### 3.2 ACL 사용자 분실

`default` 사용자가 살아 있으면 그걸로 들어가서 변경:

```bash
redis-cli -a 'default-pass'
> ACL SETUSER app_rw >NewPass!
> ACL SAVE
```

`default` 도 잠갔는데 admin 비번도 잃은 경우 — `aclfile` 직접 편집 또는 `--user default off` 끄고 재기동:

```bash
sudo systemctl stop redis
sudo vi /etc/redis/users.acl
# user default on nopass ~* +@all      ← 임시
sudo systemctl start redis

redis-cli
> ACL SETUSER admin on >NewAdmin! ~* +@all
> ACL SETUSER default off
> ACL SAVE
```

### 3.3 평문 "찾기" 는 불가 (해시 저장 시)

`ACL SETUSER user >password` 는 평문을 받지만 저장은 SHA256 해시 (`#<hex>`). 복호화 불가 — 분실하면 재설정.

> **함정**: `requirepass` 는 **평문 저장**. `redis.conf` 또는 `CONFIG GET requirepass` 로 평문 조회 가능. 그래서 더 위험.

```bash
redis-cli -a 'pass' CONFIG GET requirepass
# 1) "requirepass"
# 2) "S3cret!"     ← 평문!
```

→ `requirepass` 보다 ACL 사용 권장.

---

## 4. 외부 노출 — 가장 흔한 사고

### 4.1 `bind` + `protected-mode`

```ini
# redis.conf
bind 127.0.0.1 ::1               # localhost 만
# 또는 내부 인터페이스만
bind 127.0.0.1 10.0.1.5

protected-mode yes               # bind 없거나 requirepass 없으면 외부 거부
```

5.0+ 기본 `protected-mode yes` 가 1차 방어. **절대 끄지 말 것**.

### 4.2 사고 시나리오

```
1. Docker 로 redis 띄움 — -p 6379:6379 (0.0.0.0)
2. 방화벽 없음, requirepass 없음
3. 30 분 내 cryptojacking 봇 접속
4. CONFIG SET dir /home/user/.ssh
   CONFIG SET dbfilename authorized_keys
   SET k "<공격자-ssh-key>"
   BGSAVE
5. → SSH 로 RCE
```

방어:
1. `bind 127.0.0.1` (외부 X) 또는 Docker 는 `-p 127.0.0.1:6379:6379`
2. `requirepass` 또는 ACL
3. 방화벽 / security group 6379 inbound 제한
4. `protected-mode yes`
5. 위험 명령 비활성 (다음 절)

### 4.3 위험 명령 비활성

```ini
# redis.conf — 명령 이름 바꾸기
rename-command FLUSHALL ""
rename-command FLUSHDB  ""
rename-command CONFIG   ""
rename-command DEBUG    ""
rename-command SHUTDOWN "SHUTDOWN_a8f3"      # 길고 모르는 이름으로
rename-command KEYS     ""
rename-command MIGRATE  ""
```

`""` 는 완전 비활성. CONFIG 까지 비활성하면 운영 변경이 어려워지므로 운영 정책에 맞게.

ACL 로도 가능:
```
ACL SETUSER default -FLUSHALL -FLUSHDB -CONFIG -DEBUG -KEYS
```

---

## 5. TLS

6.0+ 부터 TLS 내장.

```ini
# redis.conf
port 0                              # 평문 포트 끄기
tls-port 6380
tls-cert-file /etc/redis/redis.crt
tls-key-file  /etc/redis/redis.key
tls-ca-cert-file /etc/redis/ca.crt
tls-auth-clients yes                # 클라이언트 인증서 강제
```

```bash
redis-cli --tls --cert client.crt --key client.key --cacert ca.crt -p 6380
```

---

## 6. 감사 / 모니터링

### 6.1 명령 로그

```bash
redis-cli MONITOR              # ⚠️ 운영에선 부담 큼 — 디버깅 임시
```

### 6.2 slowlog

```bash
redis-cli SLOWLOG GET 10
redis-cli SLOWLOG RESET
```

### 6.3 접속 추적

```bash
redis-cli CLIENT LIST
redis-cli CLIENT INFO
```

ACL 인증 실패는 로그로:

```bash
grep "ACL\|AUTH\|denied" /var/log/redis/redis-server.log
```

---

## 7. 운영 보안 체크리스트

- [ ] `bind` 는 필요한 인터페이스만 (`0.0.0.0` 금지)
- [ ] `protected-mode yes` 유지
- [ ] `requirepass` 또는 ACL — 강한 패스워드 (32+ 자, 랜덤)
- [ ] ACL 로 사용자별 권한 분리 (`default` 잠그기)
- [ ] `FLUSHALL/CONFIG/DEBUG/KEYS/MIGRATE` 비활성 또는 ACL 제한
- [ ] TLS 사용 (6.0+) — 외부 통신 시 필수
- [ ] 방화벽 / SG 로 6379 inbound 제한
- [ ] Docker 는 `-p 127.0.0.1:6379:6379` 또는 docker network 내부만
- [ ] Replication 의 `masterauth` 도 설정
- [ ] AOF / RDB 파일 권한 600
- [ ] `redis` OS 유저로 실행 (root 금지)

---

## 8. 함정 모음

### 함정 1 — Docker `-p 6379:6379`
`-p` 는 `0.0.0.0:6379` 바인드. 호스트가 인터넷에 노출되면 끝. `-p 127.0.0.1:6379:6379` 또는 internal network.

### 함정 2 — `requirepass` 평문 저장
`redis.conf` 와 `CONFIG GET` 모두 평문. 백업 / 로그에 노출 위험. **ACL + 해시 저장** 권장.

### 함정 3 — replication 의 `masterauth` 빠뜨림
master 에 `requirepass` 켰는데 slave 에 `masterauth` 없으면 replication 끊김.

```ini
# slave 의 redis.conf
masterauth S3cret!
```

### 함정 4 — `protected-mode` 끄기
"외부에서 접속이 안 돼요" 하고 `protected-mode no` 하는 순간 사고. `bind` + 방화벽으로 풀어야 함.

### 함정 5 — `MODULE LOAD` 가 열려 있음
누구나 접속하면 `MODULE LOAD /tmp/evil.so` 로 임의 코드 실행. ACL 로 `-@dangerous` 또는 `enable-module-command no` (7.x+).

### 함정 6 — Lua script 의 위험
`EVAL` 로 임의 Lua 실행. ACL 의 `-@scripting` 또는 `+EVAL +EVALSHA` 만 허용.

### 함정 7 — `ACL SAVE` 잊음
`ACL SETUSER` 는 메모리에만. `ACL SAVE` 또는 `CONFIG REWRITE` 없으면 재기동 시 사라짐.

```bash
redis-cli ACL SAVE
```

### 함정 8 — `default` 사용자 잠그기 전에 새 admin 안 만듦
ACL 작업 순서 잘못하면 자기 자신 잘림. **새 admin 먼저, 검증 후 default off**.

---

## 9. 관련

- [[configuration]] — redis.conf 일반 파라미터
- [[replication-cluster]] — masterauth / cluster auth
- [[persistence]] — RDB / AOF 파일 보안
- [[../../security/security|↗ 20-areas/security]]
- [[redis]] — Redis hub
