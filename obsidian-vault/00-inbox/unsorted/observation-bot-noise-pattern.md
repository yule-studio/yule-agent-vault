---
title: "Discord 봇 noise 패턴 관찰"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: draft
created_at: 2026-05-12T18:30:00+09:00
tags:
  - unsorted
  - inbox
---

# Discord 봇 noise 패턴 관찰

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[unsorted|↑ unsorted/]]** · **[[../inbox|↑ 00-inbox/]]**

## 관찰

Discord 멤버 봇 9 개가 한 thread 에 동시 응답하면 noise 가 큼. 운영자는 한 결정의 단일 톤을 원함.

## 근거

F15 작업 중 사용자 피드백 — "한 사람 (tech-lead) 이 외부 발화해야". decision #48 의 single-author 정책으로 이어짐.

## 정책 영향

tech-lead 만 외부 surface 발화. 6 역할 take 는 audit log 로만.

## 후속

`agents/coding/authorization.py` 의 author 출력 단일화 확인 + 회귀 테스트.

## 분류 후 이주 후보

- 정책 영향 있으면 → `decisions/` 또는 `policies/` 갱신
- 작업 진행 시작하면 → `task-logs/` 로 변환