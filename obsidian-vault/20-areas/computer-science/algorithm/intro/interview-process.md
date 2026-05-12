---
title: "기술 면접 — 프로세스 / 유형 / 준비"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T00:00:00+09:00
tags:
  - algorithm
  - interview
  - career
---

# 기술 면접 — 프로세스 / 유형 / 준비

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 — 이코테 PART 01 GUIDE 매핑 |

**[[../algorithm|↑ algorithm 인덱스]]**

## 1. 채용 프로세스 (한국 IT 기업 표준)

```
서류 (이력서 + 자소서)
  ↓
코딩 테스트 (온라인, 2-5 시간, 3-7 문제)
  ↓
1차 기술 면접 (라이브 코딩 / 알고리즘 / CS 개념)
  ↓
2차 기술 면접 (시스템 설계 / 행동 면접)
  ↓
임원 / 컬처 핏 면접
  ↓
처우 협의 / 입사
```

- 카카오 / 라인 / 쿠팡 / 토스 / 우아한형제들 등 거의 동일 구조.
- 시간: 서류 ~ 입사 평균 6-12 주.

## 2. 기술 면접 대표 유형

### (a) 알고리즘 라이브 코딩

면접관 앞에서 화이트보드 / 노트북에 코딩.
- 시간: 30-60 분 / 1-2 문제.
- 평가: 풀이 + 사고 과정 (말로 설명) + 디버깅.

### (b) CS 개념 질문

- **자료 구조**: 배열 vs 연결 리스트 / 스택 vs 큐 / 해시 충돌 / 트리 균형
- **운영체제**: 프로세스 vs 스레드 / 동기화 / 페이지 교체
- **네트워크**: TCP 3-way handshake / HTTPS / REST vs GraphQL
- **DB**: 인덱스 / 트랜잭션 / 정규화 / N+1
- **언어**: Java GC / Python GIL / C++ smart pointer

### (c) 시스템 설계 (Senior)

- "단축 URL 서비스 설계" / "트위터 피드 시스템" / "Slack 메시지 큐"
- 평가: API / DB 스키마 / 캐싱 / 확장성 / 트레이드오프

### (d) 행동 / 컬처 면접

- STAR (Situation / Task / Action / Result) 양식.
- "팀에서 갈등 해결한 경험" / "실패한 프로젝트" 등.

## 3. 준비 방법

### 알고리즘 (3-6 개월)
- 백준 / Programmers 1 일 1-2 문제 누적.
- 이코테 → LeetCode Top 150 → 백준 골드 ~ 플래티넘.
- 시간 제한 두고 풀기 (45 분 / 문제).

### CS 개념 (1-2 개월)
- [CS 공룡책 (운영체제) / 컴퓨터 네트워크 — Kurose](https://gaia.cs.umass.edu/kurose_ross/)
- [Awesome Tech Interview](https://github.com/JaeYeopHan/Interview_Question_for_Beginner) — 한국어
- 면접 빈출 질문 200 개 정리 + 1-2 분 답변 연습.

### 시스템 설계 (Senior, 2-3 개월)
- [System Design Primer](https://github.com/donnemartin/system-design-primer) (영문)
- [Grokking the System Design Interview](https://www.designgurus.io/course/grokking-the-system-design-interview)
- 5-7 대표 케이스 + 본인 스토리.

### 행동 면접 (1-2 주)
- 본인 프로젝트 5-7 개를 STAR 양식으로 정리.
- 컬쳐 핏 — 회사 가치 사전 조사 + 매칭 답변.

## 4. 면접 당일 — 라이브 코딩 5 단계

1. **문제 명확화** (5 분) — 입력 범위 / 엣지 케이스 / 출력 형식 질문.
2. **접근법 설명** (5 분) — 브루트포스 → 최적화. 시간복잡도 명시.
3. **구현** (20-30 분) — 변수 / 함수 이름 명확. 주석 최소.
4. **테스트** (5-10 분) — 작은 예 / 엣지 케이스 직접 돌려보기.
5. **개선** (남는 시간) — 시간복잡도 / 가독성 / 메모리.

**Tip**: 침묵하지 않는다. 막혔다면 "지금 고려 중인 건 X 인데..." 식으로 사고 과정 공개.

## 5. 회사별 출제 경향

| 회사 | 코테 | 면접 | 특이점 |
| --- | --- | --- | --- |
| 카카오 | 7 문제 / 5 시간 | 알고리즘 + CS | Programmers 자체 출제. 문자열 / 시뮬레이션 비율 높음 |
| 네이버 | 3-4 문제 / 3 시간 | 알고리즘 + CS | 백준 유형 / DP 비율 높음 |
| 라인 | 3 문제 / 3 시간 | 알고리즘 + CS + 시스템 설계 | 외국계 + 한국 혼합 |
| 쿠팡 | LeetCode 유형 | 영어 면접 가능 | 글로벌 빅테크 스타일 |
| 토스 | 코테 + Front Quiz | CS + 컬처 | "송금" / "보안" 도메인 |
| 우아한형제들 | 6 문제 / 3 시간 | 알고리즘 + CS + 행동 | 컬처핏 비중 큼 |
| 삼성 | SW 역량테스트 | 알고리즘 + 인적성 | 시뮬레이션 / 구현 다수 |

## 6. 안티패턴 / 함정

- **무작정 풀기** — 책 없이 풀면 비효율. 이코테 → LeetCode Patterns 순서.
- **모든 문제 외우기** — 패턴을 이해해야. 같은 알고리즘이 변형되어 나옴.
- **언어 한 개만 익숙** — 회사가 언어 강제하기도 (네이버 = Java 다수, 쿠팡 = 다양).
- **면접 당일 침묵** — 사고 과정이 평가의 70%.
- **알고리즘만 준비** — CS 개념 / 시스템 설계 / 행동 면접 모두 비중 큼.

## 7. 학습 자료

- [tech-interview-for-developer](https://github.com/gyoogle/tech-interview-for-developer) — 한국어 면접 자료 (10k+ stars)
- [Interview_Question_for_Beginner](https://github.com/JaeYeopHan/Interview_Question_for_Beginner) — 한국어 (17k+ stars)
- [동빈나 — 면접 후기 영상](https://www.youtube.com/@dongbinna) — 책 저자
- [코딩테스트 합격자 되기 (인프런)](https://www.inflearn.com/) — 한국어 강의 다수
- [Crack the Coding Interview (책)](https://www.crackingthecodinginterview.com/) — 영문 표준
- [Tech Interview Handbook](https://www.techinterviewhandbook.org/) — 글로벌 빅테크 가이드

## 관련

- [[coding-test-overview]] — 코딩 테스트 자체
- [[problem-solving-sites]] — 문제 풀이 사이트
- [[../../../../00-inbox/links/general/system-design|↗ system-design (시니어 면접용)]]
