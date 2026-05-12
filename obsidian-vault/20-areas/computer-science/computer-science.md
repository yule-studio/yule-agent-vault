---
title: "computer-science — CS 기초 / 알고리즘 / 시스템"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T01:00:00+09:00
tags:
  - computer-science
  - index
  - area
---

# computer-science — CS 영역 인덱스

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 — 6 영역 인덱스 |

> 컴퓨터 과학 (Computer Science) 의 핵심 영역을 묶은 학습 hub. 알고리즘 /
> 자료 구조 / 데이터베이스 / 네트워크 / 운영체제 / 소프트웨어 공학.
> 프로젝트와 달리 종료 시점이 없고 누적되는 지식.

**[[../areas|↑ 20-areas/]]**

## 가지 (6 영역)

| 영역 | 진입점 | 핵심 키워드 |
| --- | --- | --- |
| 🧮 **algorithm** | [[algorithm/algorithm]] | 그리디 / DFS·BFS / DP / 최단경로 / 그래프 — **이코테 사전형** |
| 📦 data-structure | [[data-structure/]] | 배열 / 연결 리스트 / 스택 큐 / 트리 / 해시 / 힙 |
| 🗄 database | [[database/]] | 인덱스 / 트랜잭션 / 정규화 / NoSQL |
| 🌐 network | [[network/]] | TCP/UDP / HTTP / TLS / DNS |
| 💻 operating-system | [[operating-system/]] | 프로세스 / 스레드 / 메모리 / 파일 시스템 |
| 🏗 software-engineering | [[software-engineering/]] | 패턴 / 리팩터링 / 테스트 / 빌드 |

## 학습 우선순위

코딩 테스트 / 신입 면접 기준:

1. **algorithm** ⭐⭐⭐⭐⭐ — 코테 통과의 필수
2. **data-structure** ⭐⭐⭐⭐⭐ — algorithm 의 전제 + 면접 필수
3. **database** ⭐⭐⭐⭐ — 백엔드 면접 필수
4. **network** ⭐⭐⭐⭐ — 모든 직군 면접
5. **operating-system** ⭐⭐⭐ — 시스템 / 백엔드
6. **software-engineering** ⭐⭐⭐ — 시니어 면접 / 실무

## 노트 표준 구조

각 영역 안의 노트는 사전 수준 14 섹션 (algorithm 처럼):
1. 한 줄 정의
2. 언제 쓰는가 (시그널 인식)
3. 핵심 직관
4. 동작 원리 (예시 데이터 깊이)
5. 복잡도 분석
6. 의사 코드
7. **언어별 구현 11 개** (Python / Java / C++ / C / JS / TS / Kotlin / Swift / Go / Rust / C#)
8. 변형 / 응용
9. 함정 / 안티패턴
10. 실전 풀이 절차
11. 대표 문제 + 풀이
12. 연습 문제 (난이도별)
13. 학습 자료
14. 관련 hub

## 운영 규칙

1. **사전이 목표** — 한 노트만 보고 그 개념을 처음부터 끝까지 이해할 수 있어야.
2. **단순 복붙 금지** — 내 해석 / 정리 / 적용 예 가 50% 이상.
3. **출처 + 날짜 필수** — 외부 자료 인용 시.
4. 같은 개념의 응용은 한 노트 안 `## 변형` 섹션.
5. 변형이 별도 알고리즘 (예: 최단 경로 안 다익스트라 / 벨만-포드 / 플로이드) 이면 서브 폴더로 분리.

## 관련

- [[../areas|↑ 20-areas/]]
- [[../../00-inbox/links/general/system-design|↗ system-design 링크 카탈로그]]
- [[../../00-inbox/links/engineering/backend/database/database|↗ database 링크 카탈로그]]
- [[../../00-inbox/links/general/books|↗ CS 책 카탈로그]]
