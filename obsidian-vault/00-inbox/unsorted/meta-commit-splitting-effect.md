---
title: "Commit 분리 정책의 효과 관찰"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: draft
created_at: 2026-05-12T18:30:00+09:00
tags:
  - unsorted
  - inbox
---

# Commit 분리 정책의 효과 관찰

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[unsorted|↑ unsorted/]]** · **[[../inbox|↑ 00-inbox/]]**

## 관찰

F14 commit splitting policy (PR ≥3 commits, --merge 우선) 도입 후 PR 리뷰 속도 개선.

## 근거

F15 사이클은 10+ commit 으로 분리됨. 각 commit 이 단일 책임 → revert 가 용이.

## 영향

다음 사이클부터 정책 그대로 유지. 사용자 메모리에 "PR 당 최소 3 commit, squash 금지" 박혀 있음.

## 비결정

CRUD operation 별 분리는 사용자가 "예시일 뿐 일률 강요 아님" 명시. 작업 성격에 따라 분리 단위 조정.

## 분류 후 이주 후보

- 정책 영향 있으면 → `decisions/` 또는 `policies/` 갱신
- 작업 진행 시작하면 → `task-logs/` 로 변환