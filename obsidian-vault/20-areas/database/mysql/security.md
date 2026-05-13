---
title: "MySQL 보안 — root 분실 / GRANT / 인증 플러그인"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:10:00+09:00
tags:
  - database
  - mysql
  - security
  - password
---

# MySQL 보안 — root 분실 / GRANT / 인증 플러그인

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | root 분실 복구 / GRANT 실수 / 외부 노출 |

**[[mysql|↑ MySQL hub]]**

> 운영자가 가장 자주 막히는 패스워드 / 권한 사고 모음.
> 일반 파라미터는 [[configuration]].

---

## 1. 패스워드 설정 — 기본

### 1.1 사용자 생성

```sql
-- MySQL 8.x
CREATE USER 'app_rw'@'10.0.%.%' IDENTIFIED BY 'S3cret!';

-- 패스워드 변경
ALTER USER 'app_rw'@'10.0.%.%' IDENTIFIED BY 'New$ecret';

-- 만료 강제 (다음 접속 시 변경 요구)
ALTER USER 'app_rw'@'10.0.%.%' PASSWORD EXPIRE;

-- 만료일 정책
ALTER USER 'app_rw'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;
```

> **함정**: MySQL 의 user 는 `'user'@'host'` 가 한 쌍. 같은 user 이름이라도 host 다르면 **다른 계정**.

### 1.2 인증 플러그인

| 플러그인 | 비고 |
| --- | --- |
| `caching_sha2_password` | ✅ 8.0+ 기본 |
| `mysql_native_password` | 구식. 오래된 클라이언트 호환용 |
| `auth_socket` | OS 유저명 = MySQL 유저명 (Linux) |
| `sha256_password` | RSA + SHA-256, TLS 필요 |

```sql
-- 확인
SELECT user, host, plugin FROM mysql.user;

-- 변경
ALTER USER 'app_rw'@'%'
  IDENTIFIED WITH caching_sha2_password BY 'NewPass!';
```

오래된 PHP / 클라이언트가 `caching_sha2_password` 못 다루면 그때만 `mysql_native_password`.

### 1.3 클라이언트 설정 파일

```ini
# ~/.my.cnf — chmod 600 필수
[client]
user=app_rw
password=S3cret!
host=db.example.com
```

```bash
chmod 600 ~/.my.cnf
mysql                    # 인자 없이도 접속
```

또는 `mysql_config_editor` (8.x) — 암호화 저장:

```bash
mysql_config_editor set --login-path=prod \
  --host=db.example.com --user=app_rw --password
mysql --login-path=prod
```

---

## 2. root 패스워드 분실 — 복구

> **전제: 서버 호스트에 OS root / sudo 권한**. 없으면 백업에서 복원해야 함.

### 2.1 방법 A — `--skip-grant-tables` (전통)

```bash
# 1. MySQL 중단
sudo systemctl stop mysql

# 2. grant table 무시하고 기동 (네트워크도 차단)
sudo mysqld_safe --skip-grant-tables --skip-networking &
# 또는 systemd override:
# sudo systemctl set-environment MYSQLD_OPTS="--skip-grant-tables --skip-networking"
# sudo systemctl start mysql

# 3. 패스워드 없이 접속
mysql -u root

# 4. 비밀번호 변경 (8.x — FLUSH PRIVILEGES 먼저)
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'NewRootPass!';
EXIT;

# 5. 정상 모드로 재기동
sudo systemctl stop mysql
sudo systemctl unset-environment MYSQLD_OPTS
sudo systemctl start mysql
```

`--skip-networking` 안 빼면 그 사이 외부 접속이 인증 없이 가능. **반드시 같이**.

### 2.2 방법 B — `init-file` (좀 더 안전)

```bash
# 1. 평문 SQL 파일 (root 만 읽기)
sudo tee /var/lib/mysql-init.sql >/dev/null <<'SQL'
ALTER USER 'root'@'localhost' IDENTIFIED BY 'NewRootPass!';
SQL
sudo chown mysql:mysql /var/lib/mysql-init.sql
sudo chmod 600 /var/lib/mysql-init.sql

# 2. 한 번만 init-file 로 기동
sudo systemctl stop mysql
sudo -u mysql mysqld --init-file=/var/lib/mysql-init.sql --skip-networking &

# 3. 끝나면 정상 기동 + 파일 삭제
sudo systemctl restart mysql
sudo rm /var/lib/mysql-init.sql
```

### 2.3 일반 사용자 패스워드 분실

root 로 들어가서 변경.

```sql
ALTER USER 'app_rw'@'%' IDENTIFIED BY 'NewPass!';
FLUSH PRIVILEGES;
```

### 2.4 패스워드 평문 "찾기" 는 불가

`mysql.user.authentication_string` 은 **해시**. 복호화 불가 — 잊으면 재설정.

```sql
SELECT user, host, authentication_string FROM mysql.user WHERE user='app_rw';
```

> "패스워드 찾아주세요" → **재설정 절차**. 평문이 어디 적혀 있다면 그게 사고.

### 2.5 `~/.my.cnf` 에 적힌 패스워드

```bash
cat ~/.my.cnf      # 평문. chmod 600 가 유일한 보호
```

비밀 관리는 `mysql_config_editor` 또는 vault.

---

## 3. GRANT — 흔한 실수

### 3.1 너무 강한 권한

```sql
-- ❌ 위험
GRANT ALL PRIVILEGES ON *.* TO 'app_rw'@'%' WITH GRANT OPTION;
```

`*.*` + `GRANT OPTION` = root 와 거의 동급. SQL injection 한 번에 DB 전체 위험.

```sql
-- ✅ 권장 — DB / 권한 좁히기
CREATE USER 'app_rw'@'10.0.%.%' IDENTIFIED BY '...';
GRANT SELECT, INSERT, UPDATE, DELETE ON app.* TO 'app_rw'@'10.0.%.%';

CREATE USER 'app_ro'@'10.0.%.%' IDENTIFIED BY '...';
GRANT SELECT ON app.* TO 'app_ro'@'10.0.%.%';

-- 마이그레이션 전용 (DDL 가능)
CREATE USER 'migrator'@'10.0.5.10' IDENTIFIED BY '...';
GRANT ALL PRIVILEGES ON app.* TO 'migrator'@'10.0.5.10';

FLUSH PRIVILEGES;
```

### 3.2 host 와일드카드 위험

```sql
-- ❌ 어디서든 접속
CREATE USER 'app_rw'@'%' IDENTIFIED BY '...';

-- ✅ 내부망만
CREATE USER 'app_rw'@'10.0.%.%' IDENTIFIED BY '...';
```

`%` 는 IP/도메인 패턴. `'app_rw'@'%'` 는 인터넷 전체.

### 3.3 권한 확인

```sql
SHOW GRANTS FOR 'app_rw'@'10.0.%.%';

-- 모든 사용자 한 번에
SELECT user, host FROM mysql.user;
SELECT * FROM information_schema.user_privileges;
```

### 3.4 `FLUSH PRIVILEGES` 빠뜨림

`GRANT` / `REVOKE` / `CREATE USER` 는 자동 반영. 하지만 `mysql.user` 테이블을 **직접 UPDATE** 했다면 `FLUSH PRIVILEGES` 필요.

```sql
-- ❌ 권장 안 함
UPDATE mysql.user SET authentication_string=PASSWORD('x') WHERE user='app_rw';
FLUSH PRIVILEGES;

-- ✅ ALTER USER
ALTER USER 'app_rw'@'%' IDENTIFIED BY 'x';
```

---

## 4. `mysql_secure_installation` — 신규 설치 후 필수

```bash
sudo mysql_secure_installation
```

체크:
- root 패스워드 설정 (없으면 강제)
- 익명 사용자 (`''@'localhost'`) 삭제
- root 의 원격 접속 차단
- `test` 데이터베이스 삭제
- 권한 테이블 reload

설치 직후 **반드시 실행**. 안 하면 익명 사용자가 그대로 남아 있을 수 있음.

```sql
-- 수동 확인
SELECT user, host FROM mysql.user WHERE user='' OR user='root';
DROP USER ''@'localhost';
DROP USER 'root'@'%';                    -- root 원격 차단
DROP DATABASE IF EXISTS test;
```

---

## 5. 외부 노출 / 네트워크 사고

### 5.1 `bind-address = 0.0.0.0`

```ini
# my.cnf
[mysqld]
bind-address = 127.0.0.1     # 외부 차단 (가장 안전)
# 또는 내부 인터페이스만
bind-address = 10.0.1.5
```

`0.0.0.0` + 방화벽 없음 + root 패스워드 약함 = 사고 공식.

### 5.2 방화벽 + IP 제한 이중

호스트의 보안 그룹 / iptables 로 3306 inbound 제한:

```bash
sudo ufw allow from 10.0.0.0/16 to any port 3306
sudo ufw deny 3306
```

사용자 host 패턴까지 좁히면 이중 방어.

### 5.3 TLS 강제

```sql
-- 사용자 단위 TLS 강제
ALTER USER 'app_rw'@'%' REQUIRE SSL;
ALTER USER 'admin'@'%' REQUIRE X509;       -- 클라이언트 인증서까지

-- 전체 강제 (서버 설정)
-- [mysqld]
-- require_secure_transport = ON
```

클라이언트:
```bash
mysql --ssl-mode=VERIFY_IDENTITY --ssl-ca=/etc/ssl/ca.pem -u app_rw -p
```

---

## 6. 감사 (audit)

### 6.1 접속 로그

```ini
# my.cnf
[mysqld]
general_log = ON                    # 운영에선 부담 큼 — 임시만
general_log_file = /var/log/mysql/general.log
log_error = /var/log/mysql/error.log
```

기본 `error.log` 에 인증 실패 남음:

```bash
grep "Access denied" /var/log/mysql/error.log
```

### 6.2 enterprise / percona audit plugin

Community 는 audit plugin 내장 X. Percona Server 의 `audit_log` 또는 MariaDB 의 `server_audit` 사용.

### 6.3 `performance_schema` 로 접속 추적

```sql
SELECT user, host, current_connections, total_connections
FROM performance_schema.accounts;
```

---

## 7. 패스워드 정책

```sql
-- 8.x — validate_password 컴포넌트
SHOW VARIABLES LIKE 'validate_password%';

SET GLOBAL validate_password.policy = STRONG;   -- LOW / MEDIUM / STRONG
SET GLOBAL validate_password.length = 12;
SET GLOBAL validate_password.mixed_case_count = 1;
SET GLOBAL validate_password.number_count = 1;
SET GLOBAL validate_password.special_char_count = 1;

-- 이력 재사용 금지
SET GLOBAL password_history = 5;            -- 최근 5개 재사용 X
SET GLOBAL password_reuse_interval = 365;   -- 365 일 내 재사용 X
```

```sql
-- 사용자 단위 만료
ALTER USER 'app_rw'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;
ALTER USER 'app_rw'@'%' FAILED_LOGIN_ATTEMPTS 5 PASSWORD_LOCK_TIME 1;
```

---

## 8. 운영 보안 체크리스트

- [ ] `mysql_secure_installation` 실행
- [ ] root 의 원격 접속 차단 (`'root'@'%'` 삭제)
- [ ] 익명 사용자 (`''@'localhost'`) 없음
- [ ] 인증 플러그인 `caching_sha2_password` (또는 그 이상)
- [ ] 앱 유저는 최소 권한 (`SELECT/INSERT/UPDATE/DELETE` 만)
- [ ] host 패턴은 내부망 CIDR 만 (`%` X)
- [ ] `bind-address` 는 필요한 인터페이스만
- [ ] `require_secure_transport = ON` (TLS 강제)
- [ ] `validate_password` + `password_history` / `password_reuse_interval`
- [ ] `FAILED_LOGIN_ATTEMPTS` + lock time
- [ ] `general_log` 은 끔, `error.log` 는 켬
- [ ] 백업 파일도 chmod 600 + 암호화 저장 ([[backup-recovery]] 참고)

---

## 9. 함정 모음

### 함정 1 — `--skip-grant-tables` + 네트워크 열림
복구 중 `--skip-networking` 빼먹으면 그 사이 누구나 root 권한으로 접속. **반드시 같이**.

### 함정 2 — `'user'@'%'` 와 `'user'@'localhost'` 가 동시에 존재
host 매칭은 specific → wildcard 순. 의도와 다른 계정으로 접속될 수 있음.

```sql
SELECT user, host, plugin FROM mysql.user WHERE user='app_rw';
-- localhost / % 양쪽 있으면 hostname 명시해서 접속
```

### 함정 3 — `GRANT` 가 비밀번호도 만든다 (구버전)
5.7 이하: `GRANT ... IDENTIFIED BY '...'` 가 동작. 8.x: 분리됨. 8.x 에서 `CREATE USER` 먼저, 그 뒤 `GRANT`.

### 함정 4 — `ALTER USER ... IDENTIFIED BY` 가 binlog 에 평문 기록
8.x 는 기본 마스킹. 5.7 + `binlog_format=STATEMENT` 는 평문 가능. row 기반 또는 8.x 권장.

### 함정 5 — `mysqldump` 가 평문 패스워드 노출
```bash
# ❌ ps 에 보임
mysqldump -u root -pSecret app > app.sql

# ✅
mysqldump --defaults-extra-file=~/.my.cnf app > app.sql
```

### 함정 6 — `caching_sha2_password` + 오래된 클라이언트
8.x default. PHP 7.x / 오래된 libmysqlclient 가 실패 — `default_authentication_plugin=mysql_native_password` 로 후퇴 말고, 클라이언트 업데이트가 정도.

### 함정 7 — `root@%` 살아 있음
설치 직후 종종 존재. **반드시 삭제**.

---

## 10. 관련

- [[configuration]] — my.cnf 일반 파라미터
- [[backup-recovery]] — dump / binlog 보안
- [[replication]] — replication 사용자 권한
- [[../../security/security|↗ 20-areas/security]]
- [[mysql]] — MySQL hub
