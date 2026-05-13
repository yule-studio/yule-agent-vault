---
title: "SQLite (Hub)"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:15:00+09:00
tags:
  - database
  - rdb
  - sqlite
  - embedded
  - hub
---

# SQLite (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | hub + 7 개 세부 노트 |

**[[../database|↑ database hub]]**

---

## 1. 한 줄 정의

**단일 파일에 저장되는 임베디드 RDB**.
2000 D. Richard Hipp — Public Domain.
세계에서 가장 많이 배포된 DB (안드로이드 / iOS / Chrome / Firefox / macOS 내부 모두 SQLite).

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 2000 | SQLite 1.0 (Hipp) — 미군 USS Oscar Austin 의 운영용 |
| 2004 | 3.0 — 새 파일 포맷 (지금까지 호환) |
| 2010 | WAL (Write-Ahead Log) 모드 |
| 2018 | JSON1 / FTS5 표준 |
| 2022 | STRICT 테이블, math 함수 |
| 2023 | 3.45 — JSON5, UUID, IIF |

---

## 3. SQLite 의 특징

1. **임베디드** — 라이브러리 (수백 KB), 서버 X
2. **단일 파일** — 모든 데이터 한 `.db` 파일
3. **Public Domain** — 라이선스 자유
4. **Cross-platform** — 어디서나 (모바일 / 데스크탑 / IoT)
5. **ACID + WAL**
6. **Dynamic Typing** — 타입 affinity (STRICT 옵션)
7. **광범위한 테스트** — 100% MC/DC coverage (항공우주급)
8. **Single Writer** — 동시 쓰기 1 (락)

---

## 4. 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[getting-started]] | 설치 / CLI / 첫 명령 |
| [[configuration]] | PRAGMA / journal mode / 옵션 |
| [[data-types]] | 타입 affinity / STRICT |
| [[sql-syntax]] | DDL / DML / 윈도우 / CTE |
| [[transactions-wal]] | ACID / WAL / journal modes |
| [[limitations]] | 한계 / 동시성 / 크기 |
| [[use-cases]] | 언제 SQLite 를 선택해야 하나 |

---

## 5. SQLite 사용 시나리오

| 시나리오 | 적합 |
| --- | --- |
| 모바일 (iOS / Android) | ✅ 표준 |
| 데스크탑 (browser / editor) | ✅ 표준 |
| IoT / 임베디드 | ✅ |
| 테스트 / 개발 환경 | ✅ |
| 데이터 분석 / 보고 / 노트북 | ✅ |
| 로컬 캐시 | ✅ |
| **작은~중간 웹사이트** (with WAL) | ✅ — 늘어남 |
| Edge / 매니지드 (Litestream, Turso) | ✅ — 새 트렌드 |
| 다중 동시 writer / 큰 운영 | ❌ — PG / MySQL |
| 네트워크 접근 | ❌ — 서버 DB |
| 분산 | ❌ — Turso / rqlite 검토 |

---

## 6. 면접 핵심 질문

1. **SQLite 가 빠른 이유** — 임베디드, 시스템콜 X.
2. **WAL** 모드 vs Rollback Journal.
3. **타입 affinity** — Dynamic Typing.
4. **단일 writer 의 함의** — busy timeout.
5. **`.db` 파일 단일** — 백업 = 파일 복사.
6. **FTS5** 전체 텍스트 검색.
7. **JSON1** 사용.
8. **STRICT 테이블**.
9. **PRAGMA foreign_keys**.
10. **Litestream / Turso** — 새 운영 방식.

---

## 7. 학습 자료

- **SQLite Documentation** — sqlite.org (최고 수준)
- **SQLite in Action** (Owens / Allen)
- **The Definitive Guide to SQLite**
- **antonz.org** — 좋은 블로그
- **DRH 의 발표** — fossil-scm

---

## 8. 관련

- [[../postgresql/postgresql]] — 큰 사촌
- [[../mysql/mysql]] — 비교
- [[../database|↑ database hub]]
