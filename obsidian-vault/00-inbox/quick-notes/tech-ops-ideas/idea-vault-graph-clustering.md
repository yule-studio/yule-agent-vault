---
title: "Vault 그래프 클러스터링 자동화"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: idea
created_at: 2026-05-12T18:30:00+09:00
tags:
  - quick_notes
  - inbox
---

# Vault 그래프 클러스터링 자동화

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[quick-notes|↑ quick-notes/]]** · **[[../inbox|↑ 00-inbox/]]**

## 아이디어

Obsidian 그래프에서 hub 별 색상 자동 부여. `_moc/<topic>` 노트 안의 [[link]] 그래프를 따라 클러스터 매핑.

## 동기

현재 그래프는 noise 가 많아 사용자가 캡처로 일부만 보고 외톨이를 찾는 형태. 자동 클러스터링이면 hub 식별이 더 빠름.

## 구현 후보

Obsidian Graph plugin 의 group 설정 + 정규식 (path: `_moc/`, color: hub-blue 등). 또는 `obsidian-graph-analysis` 커뮤니티 플러그인.

## 성공 신호

그래프 캡처 1 회로 모든 hub 가 색으로 구분 + 외톨이가 자동 강조.

## 블로커

Obsidian 설정은 사용자 vault 측 — 본 레포의 코드 변경 0.

## 분류 후 이주 후보

- 채택 시 → `10-projects/yule-studio-agent/_moc/<scope>.md` hub 에 [[link]] 추가
- 후속 PR / issue 가 land 하면 → `task-logs/` 로 변환