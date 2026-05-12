---
title: "PM Skills 카탈로그 도입"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: draft
created_at: 2026-05-12T18:30:00+09:00
tags:
  - unsorted
  - inbox
---

# PM Skills 카탈로그 도입

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[unsorted|↑ unsorted/]]** · **[[../inbox|↑ 00-inbox/]]**

## 잔여 작업

사용자가 참조 요청한 [phuryn/pm-skills](https://github.com/phuryn/pm-skills) 의 65 skills × 36 workflows × 8 domain plugins 패턴이 아직 본 레포에 미도입.

## 범위

`prompts/skills/product/` 안에 10+ skill 파일. product-manager / user-researcher / growth-analyst 가 호출 가능한 helper.

## 우선순위

F15 PR 머지 후 별도 사이클. PM 영역만 우선 → marketing 으로 확장.

## 영향

product-agent 의 prompt 가 짧아지고, skill catalog 가 단일 진실 source.

## 분류 후 이주 후보

- 정책 영향 있으면 → `decisions/` 또는 `policies/` 갱신
- 작업 진행 시작하면 → `task-logs/` 로 변환