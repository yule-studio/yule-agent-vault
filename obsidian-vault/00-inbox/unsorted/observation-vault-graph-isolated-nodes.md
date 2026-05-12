---
title: "Vault 그래프 외톨이 패턴"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: draft
created_at: 2026-05-12T18:30:00+09:00
tags:
  - unsorted
  - inbox
---

# Vault 그래프 외톨이 패턴

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[unsorted|↑ unsorted/]]** · **[[../inbox|↑ 00-inbox/]]**

## 관찰

사용자 그래프 캡처 2 회 — 외톨이 노드가 broken backlink 의 stub 인 경우가 많음.

## 근거

`[[2026-05-09_issue-73-decision-tech-lead-runtime-loop]]` 같은 작성되지 않은 노트의 stub 가 회색 노드로 나타남.

## 정책 영향

새 노트의 [[link]] 작성 시 실제 파일 존재 확인 의무화. obsidian-vault-push 가 broken link 사전 검증 후 push 가능.

## 후속

`hookify` 의 mistake signature 에 `obsidian.broken_backlink` 추가 검토.

## 분류 후 이주 후보

- 정책 영향 있으면 → `decisions/` 또는 `policies/` 갱신
- 작업 진행 시작하면 → `task-logs/` 로 변환