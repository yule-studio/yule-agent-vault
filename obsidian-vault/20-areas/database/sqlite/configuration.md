---
title: "SQLite 설정 — PRAGMA / Journal Mode / 옵션"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:25:00+09:00
tags:
  - database
  - sqlite
  - configuration
  - pragma
---

# SQLite 설정 — PRAGMA

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 핵심 PRAGMA |

**[[sqlite|↑ SQLite hub]]**

---

## 1. PRAGMA 란?

SQLite 의 모든 설정은 SQL 처럼 `PRAGMA` 로:

```sql
PRAGMA key = value;       -- 설정
PRAGMA key;               -- 조회
```

⚠️ 대부분 **연결 단위** — 매 connection 마다 설정.

---

## 2. journal_mode

```sql
PRAGMA journal_mode = WAL;
```

| 모드 | 의미 |
| --- | --- |
| `DELETE` (기본) | journal 파일 삭제 — 표준 rollback |
| `TRUNCATE` | journal 헤더만 trun |
| `PERSIST` | journal 유지 (다시 사용) |
| `MEMORY` | journal 메모리 — crash 시 손상 |
| **`WAL`** | Write-Ahead Log — 동시 read+write |
| `OFF` | journal 없음 — 데이터 손상 위험 |

**WAL 권장** — 자세히 → [[transactions-wal]]

---

## 3. synchronous

```sql
PRAGMA synchronous = NORMAL;
```

| 값 | 동작 | 안전 | 성능 |
| --- | --- | --- | --- |
| `OFF` (0) | fsync 안 함 | ★ — crash 시 손상 | ★★★ |
| `NORMAL` (1) | WAL 에 자주, 다른 모드에 매 commit | ★★ | ★★ |
| `FULL` (2, 기본) | 매 commit fsync | ★★★ | ★ |
| `EXTRA` (3) | + 디렉터리 동기 | ★★★ | 더 느림 |

WAL + `NORMAL` 이 표준 (안전 + 빠름).

---

## 4. foreign_keys

```sql
PRAGMA foreign_keys = ON;
```

⚠️ 기본 OFF. **매 연결** 활성 필요. ORM 이 자동인 경우도.

---

## 5. busy_timeout

```sql
PRAGMA busy_timeout = 5000;        -- ms
```

다른 트랜잭션의 lock 으로 SQLITE_BUSY 발생 → 5초 대기 후 에러.
다중 writer 환경에 필수.

---

## 6. cache_size

```sql
PRAGMA cache_size = -64000;        -- 음수 = KB (64MB)
PRAGMA cache_size = 10000;         -- 양수 = page 수
```

기본 `-2000` = 2MB. 큰 워크로드는 더 크게.

---

## 7. mmap_size

```sql
PRAGMA mmap_size = 30000000000;    -- 30GB
```

파일을 메모리에 mmap. 큰 DB 의 read 가속 (OS 캐시).

---

## 8. temp_store

```sql
PRAGMA temp_store = MEMORY;        -- 임시 인덱스 / 정렬 메모리
```

| 값 | 의미 |
| --- | --- |
| `DEFAULT` | compile-time |
| `FILE` | 디스크 |
| `MEMORY` | 메모리 |

---

## 9. page_size / encoding

```sql
PRAGMA page_size = 8192;          -- 새 DB 에서만, 기본 4096
PRAGMA encoding = "UTF-8";        -- UTF-8 / UTF-16le / UTF-16be
```

→ `VACUUM` 으로 기존 DB 의 page_size 변경 가능.

---

## 10. auto_vacuum

```sql
PRAGMA auto_vacuum = FULL;        -- NONE / FULL / INCREMENTAL
```

| 모드 | 의미 |
| --- | --- |
| `NONE` (기본) | DELETE 후 공간 회수 X (VACUUM 수동) |
| `FULL` | 매 트랜잭션 회수 |
| `INCREMENTAL` | 명시적 `PRAGMA incremental_vacuum` |

자주 삭제하는 워크로드면 INCREMENTAL 권장.

---

## 11. analysis_limit / optimize

```sql
PRAGMA analysis_limit = 1000;
PRAGMA optimize;                    -- 통계 갱신 (3.18+)
```

ANALYZE 와 비슷 — 옵티마이저 결정 개선. 종료 시 한 번 권장.

---

## 12. defer_foreign_keys

```sql
PRAGMA defer_foreign_keys = ON;     -- 트랜잭션 끝까지 검사 연기
```

순환 참조 / bulk 삽입 시.

---

## 13. case_sensitive_like

```sql
PRAGMA case_sensitive_like = ON;
```

기본 OFF — `LIKE` 가 대소문자 무시. SQL 표준은 sensitive.

---

## 14. recursive_triggers

```sql
PRAGMA recursive_triggers = ON;
```

트리거가 다른 트리거 호출 가능.

---

## 15. legacy_alter_table

```sql
PRAGMA legacy_alter_table = OFF;    -- 기본 OFF (3.25+)
```

새 ALTER 동작이 더 안전 (view 등 자동 갱신).

---

## 16. trusted_schema

```sql
PRAGMA trusted_schema = OFF;
```

application_id 와 함께 보안 (사용자 입력 DB 검사).

---

## 17. 디스크 / 격리

### 17.1 locking_mode

```sql
PRAGMA locking_mode = EXCLUSIVE;
```

`NORMAL` (기본) / `EXCLUSIVE` — exclusive 는 한 connection 만, 단일 프로세스 워크로드 가속.

### 17.2 read_uncommitted (shared cache 모드)

```sql
PRAGMA read_uncommitted = 1;
```

같은 connection / shared cache 안에서. 일반 SQLite 에선 의미 적음.

---

## 18. 권장 시작 PRAGMA (대부분의 경우)

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA foreign_keys = ON;
PRAGMA busy_timeout = 5000;
PRAGMA cache_size = -65536;          -- 64MB
PRAGMA temp_store = MEMORY;
PRAGMA mmap_size = 268435456;        -- 256MB
PRAGMA optimize;                      -- 시작 / 종료 시
```

### 18.1 모바일 / 리소스 제약

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA foreign_keys = ON;
PRAGMA cache_size = -4000;           -- 4MB
PRAGMA mmap_size = 0;                 -- 메모리 mmap 제한
```

### 18.2 분석 / 노트북

```sql
PRAGMA journal_mode = OFF;            -- 휘발 OK
PRAGMA synchronous = OFF;
PRAGMA temp_store = MEMORY;
PRAGMA cache_size = -2000000;         -- 2GB
```

---

## 19. VACUUM

```sql
VACUUM;                              -- 전체 재작성 (한 번에 대기)
VACUUM INTO 'compact.db';            -- 다른 파일로
PRAGMA incremental_vacuum(1000);     -- 페이지 단위
```

DELETE 후 디스크 회수 / page 정리.

---

## 20. 진단

```sql
PRAGMA database_list;
PRAGMA table_info(users);
PRAGMA index_list(users);
PRAGMA index_info(idx_email);
PRAGMA foreign_key_list(orders);
PRAGMA integrity_check;
PRAGMA quick_check;
PRAGMA wal_checkpoint(TRUNCATE);
PRAGMA freelist_count;
PRAGMA page_count;
PRAGMA page_size;
```

---

## 21. Compile-time Options

라이브러리 빌드 시 `-D` 로:
- `SQLITE_ENABLE_FTS5`
- `SQLITE_ENABLE_JSON1`
- `SQLITE_THREADSAFE=1/2`
- `SQLITE_DEFAULT_PAGE_SIZE`
- `SQLITE_MAX_LENGTH`

대부분 배포 라이브러리는 표준 활성.

---

## 22. 함정

### 22.1 FK 기본 OFF
가장 흔한 실수. 매 연결 ON.

### 22.2 WAL 의 -shm / -wal 파일
DB 옆에 `app.db-wal`, `app.db-shm`. 백업 시 같이 / checkpoint 후 복사.

### 22.3 `synchronous = OFF`
crash 시 손상. 분석 / 임시 DB 에만.

### 22.4 mmap_size 너무 큼
프로세스 가상 메모리 ↑. 32-bit 시스템에서 한계.

### 22.5 PRAGMA 가 SET 안 되는 경우
일부 PRAGMA 는 트랜잭션 안에서 변경 X.

### 22.6 connection 풀의 PRAGMA 불일치
풀 가져올 때마다 같은 PRAGMA 설정 (또는 onConnect hook).

### 22.7 cache_size 의 음수 / 양수
음수 = KB, 양수 = page. 혼동 주의.

---

## 23. 학습 자료

- **PRAGMA Statements** — sqlite.org/pragma.html
- **WAL Mode** — sqlite.org/wal.html
- **antonz.org blog** — 좋은 PRAGMA 가이드

---

## 24. 관련

- [[transactions-wal]] — WAL / journal
- [[sqlite]] — SQLite hub
