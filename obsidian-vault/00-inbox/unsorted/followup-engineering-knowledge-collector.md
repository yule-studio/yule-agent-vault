---
title: "Engineering knowledge 자동 수집기"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: draft
created_at: 2026-05-12T18:30:00+09:00
tags:
  - unsorted
  - inbox
---

# Engineering knowledge 자동 수집기

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[unsorted|↑ unsorted/]]** · **[[../inbox|↑ 00-inbox/]]**

## 잔여 작업

F5/F13 의 부서별 자동 자료 수집 (RSS crawler + dispatch) 가 wiring 만 되어 있고 실 운영 검증 미완료.

## 범위

`agents/engineering_intelligence/` — 16 host RSS / release feed → dept_router → dedup ledger → Discord 카드.

## 우선순위

사용자 호스트에서 `yule runtime up` 후 `eng-digest-scheduler` 서비스 활성화 확인.

## 영향

각 부서의 daily digest 가 자동 게시 → 운영자 수동 수집 부담 0.

## 분류 후 이주 후보

- 정책 영향 있으면 → `decisions/` 또는 `policies/` 갱신
- 작업 진행 시작하면 → `task-logs/` 로 변환