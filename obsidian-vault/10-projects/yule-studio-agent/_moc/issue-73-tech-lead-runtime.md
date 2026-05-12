---
title: "MOC — tech-lead Runtime Loop (#73)"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: completed
created_at: 2026-05-12T15:30:00+09:00
tags:
  - moc
  - hub
  - tech-lead-runtime
  - issue-73
related:
  - _moc.md
---

# MOC — tech-lead Runtime Loop

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 hub |

## 이 hub 가 묶는 것

issue #73 의 tech-lead runtime loop 4 단계 (coding executor /
completion hook + selector / decision layer / 검증) 의 모든 노트.

**[[_moc|↑ MOC 인덱스]]**

## 결정 / 작업 로그

- [[../task-logs/task-log-tech-lead-runtime-loop-issue-73]] — 12 결정
  (D-73-1 ~ D-73-12) + 4 단계 진행
- [[../decisions/decision-tech-lead-single-write-subject-issue-48]] —
  tech-lead 의 단일 write 주체 정책 (이 runtime loop 의 author 모델)

## 4 단계 요약

1. coding executor (worktree → edit → test → commit → push → draft PR)
2. completion hook + selector (역할 선정 / 가중치)
3. decision layer (autonomy_policy L0~L4 + governance 4 layer)
4. 검증 (회귀 / e2e / supervisor)

## 관련 hub

- [[hermes-yule-integration]] — 본 runtime loop 의 패턴 입력
- [[f15-corporate-structure]] — 본 loop 가 부서 단위로 확장된 모습
