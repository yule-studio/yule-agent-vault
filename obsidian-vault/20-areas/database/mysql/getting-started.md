---
title: "MySQL 시작하기 — 설치 / 초기화 / 첫 연결"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:15:00+09:00
tags:
  - database
  - mysql
  - setup
---

# MySQL 시작하기

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 설치 / 초기화 / 첫 연결 |

**[[mysql|↑ MySQL hub]]**

---

## 1. 설치

### 1.1 macOS

```bash
brew install mysql
brew services start mysql

mysql_secure_installation   # root 비밀번호 + 익명 사용자 제거
```

### 1.2 Ubuntu

```bash
sudo apt update
sudo apt install mysql-server-8.0

sudo systemctl enable --now mysql
sudo mysql_secure_installation
```

### 1.3 Docker

```bash
docker run -d \
  --name mysql8 \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=app \
  -e MYSQL_USER=app \
  -e MYSQL_PASSWORD=apppw \
  -p 3306:3306 \
  -v mysql8-data:/var/lib/mysql \
  mysql:8.0
```

### 1.4 매니지드 / 변형

| 서비스 | 특징 |
| --- | --- |
| AWS RDS MySQL | 표준 RDS |
| AWS Aurora MySQL | 분산 스토리지 |
| GCP Cloud SQL | 매니지드 |
| **PlanetScale** | Vitess 기반 서버리스 |
| **MariaDB** | Monty 의 fork |
| **Percona Server** | Oracle MySQL + 패치 |
| **TiDB** | MySQL 호환 분산 |

---

## 2. 디렉터리

```
/var/lib/mysql            # 데이터 (Linux)
/etc/mysql/my.cnf         # 설정
/var/log/mysql/           # 로그
/opt/homebrew/var/mysql/  # macOS brew
```

---

## 3. 첫 접속

```bash
mysql -h localhost -u root -p
mysql -u root -p app
mysql --defaults-file=~/.my.cnf
```

### 3.1 ~/.my.cnf

```ini
[client]
host=localhost
user=app
password=secret
database=app
```
권한 `chmod 600 ~/.my.cnf`.

---

## 4. mysql CLI 필수 명령

```sql
SHOW DATABASES;
USE app;
SHOW TABLES;
DESC users;             -- 또는 DESCRIBE users
SHOW CREATE TABLE users\G
SHOW INDEXES FROM users;
SHOW PROCESSLIST;       -- 활성 connection
SHOW STATUS;            -- 전역 상태
SHOW VARIABLES LIKE 'innodb%';
\s                       -- 서버 정보
\q                       -- 종료

-- \G — 세로 출력 (긴 줄에 좋음)
SELECT * FROM users\G
```

---

## 5. 사용자 / DB 만들기

```sql
CREATE DATABASE app CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;

CREATE USER 'app'@'%' IDENTIFIED BY 'secret';
GRANT ALL PRIVILEGES ON app.* TO 'app'@'%';
FLUSH PRIVILEGES;

SHOW GRANTS FOR 'app'@'%';
```

### 5.1 권한 분리 (실무)

```sql
-- 읽기/쓰기
CREATE USER 'app_rw'@'%' IDENTIFIED BY '...';
GRANT SELECT, INSERT, UPDATE, DELETE ON app.* TO 'app_rw'@'%';

-- 읽기 전용
CREATE USER 'app_ro'@'%' IDENTIFIED BY '...';
GRANT SELECT ON app.* TO 'app_ro'@'%';

-- DDL (마이그레이션)
CREATE USER 'app_ddl'@'%' IDENTIFIED BY '...';
GRANT ALL ON app.* TO 'app_ddl'@'%';
```

### 5.2 호스트 패턴

```
'app'@'localhost'    -- 같은 머신
'app'@'10.0.0.%'     -- 같은 서브넷
'app'@'%'            -- 모두
```

---

## 6. 첫 테이블

```sql
CREATE TABLE users (
    id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    email       VARCHAR(320) NOT NULL,
    name        VARCHAR(100),
    created_at  DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    PRIMARY KEY (id),
    UNIQUE KEY uk_users_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

INSERT INTO users (email, name)
VALUES ('alice@example.com', 'Alice'),
       ('bob@example.com',   'Bob');

SELECT * FROM users;
```

---

## 7. 연결 문자열

```
mysql://user:pass@host:port/dbname?param=val

# 예
mysql://app:secret@localhost:3306/app
mysql+pymysql://app:secret@db.example.com/app?ssl=true&charset=utf8mb4
```

---

## 8. 문자 인코딩 — utf8 vs utf8mb4

⚠️ MySQL 의 `utf8` 은 **3 byte 까지만** (이모지 X). **항상 `utf8mb4`** 를 사용.

```sql
ALTER DATABASE app CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
ALTER TABLE users CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

```ini
# my.cnf
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_0900_ai_ci

[client]
default-character-set = utf8mb4
```

### 8.1 Collation

| Collation | 의미 |
| --- | --- |
| `utf8mb4_0900_ai_ci` | MySQL 8.0+ 기본, accent/case insensitive |
| `utf8mb4_unicode_ci` | 구식, 잘못된 정렬 가능 |
| `utf8mb4_bin` | 바이너리 비교 (정확) |
| `utf8mb4_ko_0900_as_cs` | 한국어 + accent/case sensitive |

---

## 9. 시간대

```sql
SHOW VARIABLES LIKE '%time_zone%';
SET time_zone = '+00:00';   -- UTC

-- 또는 my.cnf
default-time-zone = '+00:00'
```

저장은 UTC, 표시는 클라이언트에서 변환 권장.

---

## 10. 함정

### 함정 1 — root 계정으로 앱 연결
사고의 시작. 권한 분리.

### 함정 2 — `utf8` (3 byte)
이모지 / 일부 한자 깨짐. **`utf8mb4`** 필수.

### 함정 3 — `MyISAM` 사용
트랜잭션 X, crash 시 데이터 손상. **InnoDB** 가 표준.

### 함정 4 — `ONLY_FULL_GROUP_BY`
MySQL 5.7+ 기본. `GROUP BY id` 후 `SELECT name` 에러. SQL 표준 따름.

### 함정 5 — `--skip-grant-tables`
비밀번호 우회 모드. 운영 절대 X.

### 함정 6 — 3306 외부 노출
방화벽 / SG. SSL 강제.

---

## 11. 관련

- [[configuration]] — my.cnf
- [[sql-syntax]] — SQL
- [[mysql]] — MySQL hub
