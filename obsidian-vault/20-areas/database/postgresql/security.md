---
title: "PostgreSQL 보안 — 패스워드 / 권한 / 분실 복구"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:00:00+09:00
tags:
  - database
  - postgresql
  - security
  - password
---

# PostgreSQL 보안 — 패스워드 / 권한 / 분실 복구

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 패스워드 설정 / 분실 복구 / 권한 흔한 실수 |

**[[postgresql|↑ PostgreSQL hub]]**

> 이 노트는 **운영자가 가장 자주 막히는 보안 시나리오** 모음.
> 설정 파라미터 전반은 [[configuration]], 인증 method 선택은 거기 §3 참고.

---

## 1. 패스워드 설정 — 기본

### 1.1 사용자 생성 + 패스워드

```sql
-- 새 role 만들면서 패스워드
CREATE ROLE app_rw WITH LOGIN PASSWORD 'S3cret!';

-- 기존 role 패스워드 변경
ALTER ROLE app_rw WITH PASSWORD 'New$ecret';

-- 만료일 같이
ALTER ROLE app_rw VALID UNTIL '2026-12-31';
```

`CREATE USER` = `CREATE ROLE ... LOGIN`. 차이 그뿐.

### 1.2 패스워드 해시 알고리즘

```sql
SHOW password_encryption;     -- scram-sha-256 (v14+ 기본)
```

`md5` 는 **사용 금지**. `password_encryption = scram-sha-256` 으로 두고 `ALTER ROLE ... PASSWORD` 다시 실행해야 SCRAM 으로 재저장됨.

```sql
-- 어떤 사용자가 아직 md5 로 저장돼 있나
SELECT rolname, rolpassword FROM pg_authid WHERE rolpassword LIKE 'md5%';
```

### 1.3 환경 변수 / .pgpass 로 noninteractive 접속

```bash
# ~/.pgpass — chmod 600 필수
# hostname:port:database:username:password
localhost:5432:*:app_rw:S3cret!

# 또는 환경변수
export PGPASSWORD='S3cret!'
psql -h db.example.com -U app_rw -d app
```

`PGPASSWORD` 는 process list 에 노출될 수 있어 **CI/스크립트 외 비추천**. 운영은 `.pgpass` + `chmod 600`.

---

## 2. 패스워드를 잊어버렸을 때 — 복구

### 2.1 superuser 패스워드 분실 (가장 흔함)

> **전제: 서버 호스트에 OS 접근 권한이 있다.** OS 접근이 없으면 복구 불가 — 백업에서 복원해야 함.

**방법 A — `pg_hba.conf` 를 `trust` 로 임시 변경**

```bash
# 1. pg_hba.conf 백업
sudo cp $PGDATA/pg_hba.conf $PGDATA/pg_hba.conf.bak

# 2. local 라인을 trust 로 (또는 맨 위에 한 줄 추가)
sudo vi $PGDATA/pg_hba.conf
# local   all   postgres   trust   ← 임시

# 3. reload (restart 아님)
sudo systemctl reload postgresql
# 또는
sudo -u postgres pg_ctl reload -D $PGDATA

# 4. 패스워드 없이 접속 → 변경
sudo -u postgres psql
postgres=# ALTER USER postgres WITH PASSWORD 'NewS3cret!';

# 5. pg_hba.conf 원복 + reload
sudo cp $PGDATA/pg_hba.conf.bak $PGDATA/pg_hba.conf
sudo systemctl reload postgresql
```

**방법 B — `peer` (Linux 기본)**

기본 설치는 `local all postgres peer` — OS `postgres` 유저면 패스워드 없이 들어감.

```bash
sudo -u postgres psql
postgres=# \password postgres   # 대화형 패스워드 변경
```

`peer` 가 안 되면 `pg_hba.conf` 에 `local all postgres peer` 라인이 있는지 확인.

### 2.2 일반 사용자 패스워드 분실

superuser 로 들어가서 그냥 변경.

```sql
ALTER USER app_rw WITH PASSWORD 'NewPass!';
```

### 2.3 평문 패스워드 "찾기" 는 불가능

`pg_authid.rolpassword` 는 SCRAM/MD5 **해시**. 복호화 불가. 분실하면 **새로 설정** 하는 길뿐.

```sql
-- 해시만 확인 가능 (superuser 만)
SELECT rolname, rolpassword FROM pg_authid WHERE rolname = 'app_rw';
-- SCRAM-SHA-256$4096:...$...:...
```

> **함정**: "패스워드 찾아주세요" 요청 받으면 **재설정 절차** 로 안내. 평문을 들고 다니는 시스템이라면 그게 사고.

### 2.4 `~/.pgpass` 에 저장한 패스워드 보기

```bash
cat ~/.pgpass        # 평문 (chmod 600 이 유일한 보호)
```

`.pgpass` 에 적은 패스워드는 평문이라 "찾을" 수 있지만, 그 자체가 보안 약점. 비밀 관리는 vault (1Password / Vault / AWS Secrets Manager) 로.

---

## 3. 권한 — 흔한 실수

### 3.1 모든 권한을 app 유저에 주는 실수

```sql
-- ❌ 위험
GRANT ALL PRIVILEGES ON DATABASE app TO app_rw;
ALTER USER app_rw WITH SUPERUSER;
```

운영 앱이 superuser 면 SQL injection 한 번에 `DROP DATABASE`. **앱은 최소 권한**.

```sql
-- ✅ 권장 — DDL/DML 분리
CREATE ROLE app_rw LOGIN PASSWORD '...';
CREATE ROLE app_ro LOGIN PASSWORD '...';

GRANT CONNECT ON DATABASE app TO app_rw, app_ro;
GRANT USAGE ON SCHEMA public TO app_rw, app_ro;

GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_rw;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_ro;

-- 앞으로 만들어질 테이블에도 자동 부여
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;
```

### 3.2 `public` schema 의 기본 권한 함정

v15 이전: 누구나 `public` 에 테이블 생성 가능. v15+ 부터는 막혀 있음.

```sql
-- v14 이하라면 명시적으로 제거
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE app FROM PUBLIC;
```

### 3.3 `GRANT` 가 안 먹는 것 같을 때

```sql
-- 권한 확인
\du app_rw                              -- role 속성
\dp public.users                        -- 객체 권한

SELECT * FROM information_schema.role_table_grants
WHERE grantee = 'app_rw';
```

자주 보는 원인:
- `USAGE ON SCHEMA` 빼먹음 → 테이블 권한 있어도 안 보임
- `GRANT` 는 새로 만든 테이블엔 자동 안 됨 → `ALTER DEFAULT PRIVILEGES` 필요
- `public` schema 가 아닌데 `search_path` 가 `public` 만 → 객체를 못 찾음

---

## 4. pg_hba.conf — 자주 막히는 사고

### 4.1 "FATAL: no pg_hba.conf entry for host …"

원인: 클라이언트 IP 가 어떤 라인에도 매치 안 됨. 또는 사용자/DB 매칭 안 됨.

```bash
# 클라이언트 IP 확인 후 pg_hba.conf 에 추가
host    app    app_rw    10.0.5.0/24    scram-sha-256

# reload
SELECT pg_reload_conf();   -- psql
sudo systemctl reload postgresql
```

### 4.2 "FATAL: password authentication failed"

원인 후보:
- 진짜 비밀번호 틀림
- SCRAM 으로 저장된 사용자에 md5-only 클라이언트가 접근 (오래된 libpq)
- `password_encryption` 바꾼 뒤 `ALTER ROLE ... PASSWORD` 재실행 안 함

```sql
-- 사용자의 해시 알고리즘 확인
SELECT rolname, substring(rolpassword for 14) FROM pg_authid WHERE rolname='app_rw';
-- 'SCRAM-SHA-256$' 또는 'md5...'
```

### 4.3 reload 안 하고 헤매기

`pg_hba.conf` 변경은 **reload 필수**. restart 아님.

```sql
SELECT pg_reload_conf();
```

---

## 5. 외부 노출 / 네트워크 사고

### 5.1 `listen_addresses = '*'` + 방화벽 없음

PostgreSQL 5432 가 인터넷에 열림 → brute force 시작. Shodan 으로 검색되는 사고 사례 많음.

**최소 방어**:
1. `listen_addresses` 는 내부 인터페이스 / VPC 만
2. 방화벽 (iptables / security group) 으로 5432 inbound 제한
3. `pg_hba.conf` 에서 `0.0.0.0/0` 매치 라인 없애기
4. SSL 강제 — `ssl = on` + `hostssl` 만

```ini
# postgresql.conf
listen_addresses = 'localhost,10.0.1.5'   # 외부 X
ssl = on
ssl_cert_file = '/etc/ssl/postgres/server.crt'
ssl_key_file = '/etc/ssl/postgres/server.key'
```

```
# pg_hba.conf — hostssl 만 허용
hostssl  all  all  10.0.0.0/16  scram-sha-256
host     all  all  0.0.0.0/0    reject
```

### 5.2 SSL 강제하기

```sql
-- 사용자 단위
ALTER ROLE app_rw SET ssl_require = on;       -- 이건 안 됨, 아래 방식
```

서버 측에서 강제:
- `pg_hba.conf` 에 `host` 대신 `hostssl` 만 쓰면 비-SSL 거부
- `host all all 0.0.0.0/0 reject` 를 비-SSL 매치 위로

클라이언트:
```bash
psql "host=db.example.com sslmode=verify-full sslrootcert=/etc/ssl/ca.crt user=app_rw"
```

---

## 6. 감사 (audit)

### 6.1 접속/실패 로그

```ini
# postgresql.conf
log_connections = on
log_disconnections = on
log_hostname = off                # DNS 조회 비용 X
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '
```

### 6.2 실패 패스워드 로그에서 잡기

```bash
grep "authentication failed" /var/log/postgresql/postgresql-*.log
grep "no pg_hba.conf entry" /var/log/postgresql/postgresql-*.log
```

같은 IP 가 빠르게 반복되면 fail2ban / iptables 로 차단.

### 6.3 `pgaudit` 익스텐션

```sql
CREATE EXTENSION pgaudit;
-- postgresql.conf
-- shared_preload_libraries = 'pgaudit'
-- pgaudit.log = 'write, ddl'
```

DDL / write 만 로그 남기면 운영 부담 적음.

---

## 7. 패스워드 정책

PostgreSQL 자체엔 강한 패스워드 정책 (길이/만료/이력) **내장 없음**. `passwordcheck` 확장 정도.

```sql
-- postgresql.conf
-- shared_preload_libraries = 'passwordcheck'
CREATE EXTENSION IF NOT EXISTS passwordcheck;
-- 짧거나 사용자명과 같으면 거부
```

실무는:
- LDAP / Kerberos / OIDC 연동 → 정책은 IdP 에서
- 만료는 `ALTER ROLE ... VALID UNTIL`
- 패스워드 자체보다 **IP 제한 + TLS 클라이언트 인증서** 가 강함

```sql
ALTER ROLE app_rw VALID UNTIL '2026-12-31';   -- 만료
ALTER ROLE app_rw CONNECTION LIMIT 50;        -- 동시 접속 제한
```

---

## 8. 운영 보안 체크리스트

- [ ] superuser 패스워드 vault 에만 저장
- [ ] `password_encryption = scram-sha-256` + 모든 사용자 SCRAM 으로 재저장
- [ ] 앱 유저는 최소 권한 (SELECT/INSERT/UPDATE/DELETE 만)
- [ ] `public` schema CREATE 권한 PUBLIC 회수
- [ ] `pg_hba.conf` 에 `trust` 없음 (테스트 외)
- [ ] `listen_addresses` 는 필요한 인터페이스만
- [ ] `ssl = on` + `hostssl` 만 허용
- [ ] `log_connections / log_disconnections` 켜기
- [ ] `statement_timeout` / `idle_in_transaction_session_timeout` 설정
- [ ] 정기 `pg_stat_activity` 검토 (장기 idle 트랜잭션)
- [ ] 백업 파일도 권한 600 + 암호화 저장
- [ ] 패스워드 만료 (`VALID UNTIL`) 또는 IdP 연동

---

## 9. 함정 모음

### 함정 1 — `trust` 를 잠깐만 켜고 잊음
복구 후 `pg_hba.conf` 원복 안 하면 그대로 인터넷에 열림.

### 함정 2 — `md5` 알고리즘
v14+ 기본은 SCRAM 이지만, 업그레이드한 클러스터의 기존 사용자는 여전히 md5 일 수 있음. `ALTER ROLE ... PASSWORD` 한 번 다시 실행.

### 함정 3 — 환경 변수 `PGPASSWORD`
process list (`ps auxe`) 에 노출. `.pgpass` 권장.

### 함정 4 — 백업 파일 안에 패스워드
`pg_dumpall --globals-only` 는 SCRAM 해시 포함. 백업 파일도 chmod 600 + 암호화 저장.

### 함정 5 — `ALTER USER ... PASSWORD '...'` 가 `pg_stat_statements` 에 남음
v15+ 는 자동 마스킹되지만, 구버전은 평문 패스. `\password` 메타 명령 (대화형) 권장.

```sql
-- ❌ pg_stat_statements 에 남을 수 있음
ALTER USER app_rw WITH PASSWORD 'plain';

-- ✅ psql 메타 명령 — 클라이언트에서 해시 후 전송
\password app_rw
```

### 함정 6 — replication 슬레이브에 인증 안 걸기
streaming replication 의 `replication` 사용자도 SCRAM + IP 제한 필수. 안 그러면 누구나 WAL 받아감.

---

## 10. 관련

- [[configuration]] — pg_hba.conf 전체 구조 / authentication method
- [[backup-recovery]] — 백업 파일 보안
- [[replication]] — replication 유저 권한
- [[../../security/security|↗ 20-areas/security]] — 일반 보안 원칙
- [[postgresql]] — PostgreSQL hub
