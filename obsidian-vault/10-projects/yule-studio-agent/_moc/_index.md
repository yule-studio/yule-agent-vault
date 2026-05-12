---
title: "yule-studio-agent — Map of Content (전체 인덱스)"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T15:00:00+09:00
tags:
  - moc
  - index
  - hub
  - knowledge
related:
  - ../README.md
---

# yule-studio-agent vault 의 모든 길

Obsidian 그래프가 헝클어지지 않도록, 모든 노트는 다음 hub 중 하나에서
시작해 따라간다. 각 hub 는 한 주제의 decision · research · task-log ·
knowledge 를 묶는 단일 진입점.

## 주요 hub

### F15 — 회사 구조 + manifest 통일 (2026-05-11 ~ 2026-05-12)

- [[f15-corporate-structure]] — 부서 골격 (CTO/CPO/CMO) + 의도적 분리
- [[manifest-migration]] — agent.json → manifest.json 통일 (4 commit)
- [[plugins-catalog]] — 11 플러그인 운영 카탈로그

### 운영 인프라 (#69 ~ #126)

- [[issue-73-tech-lead-runtime]] — tech-lead runtime loop 4 단계
- [[discord-ci-strategy]] — Discord ↔ CI 경계 결정
- [[engineering-knowledge-surface]] — 자료 share scope + retrieval

### 외부 통합

- [[hermes-yule-integration]] — Hermes agent → Yule 흡수 결정

## 폴더 별 인덱스

- [[../decisions/decisions|decisions/]] — 모든 결정 노트
- [[../research/research|research/]] — 모든 리서치 노트
- [[../task-logs/task-logs|task-logs/]] — 모든 작업 로그
- [[../knowledge/knowledge|knowledge/]] — 운영 지식
- [[../knowledge/plugins/plugins|knowledge/plugins/]] — 플러그인 운영
- [[../yule-studio-agent|↑ 프로젝트 진입점]]

## 새 노트 추가 시 체크리스트

1. 명명 컨벤션 (`<kind>-<topic>[-issue-<n>].md`) 준수 (날짜 prefix 금지)
2. 표준 frontmatter (title / kind / project / agent / status /
   created_at / related)
3. 본 hub 의 적절한 카테고리 아래 [[link]] 추가
4. 같은 주제의 다른 단계 노트 (decision/research/task-log triplet) 와
   frontmatter `related` 로 cross-link
