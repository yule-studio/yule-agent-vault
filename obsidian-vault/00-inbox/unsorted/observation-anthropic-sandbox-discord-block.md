---
title: "Anthropic sandbox 의 Discord 차단"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: draft
created_at: 2026-05-12T18:30:00+09:00
tags:
  - unsorted
  - inbox
---

# Anthropic sandbox 의 Discord 차단

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[unsorted|↑ unsorted/]]** · **[[../inbox|↑ 00-inbox/]]**

## 관찰

본 Claude Code 세션에서 `yule doctor` 가 `FAIL discord tls ... Forbidden`. macOS sandbox-exec 룰이 outbound TLS 를 제한.

## 영향

sandbox 안에서는 Discord 봇 attach 불가. 운영 검증은 사용자 호스트 (`yule discord up` / `yule runtime up`) 에서만.

## 정책

sandbox 차단을 안전 기본값으로 수용. 우회 시도 금지 — Anthropic 의 의도된 격리.

## 대안

dry-run + e2e CLI smoke (intake → approve → progress → complete) 로 startup path 검증.

## 분류 후 이주 후보

- 정책 영향 있으면 → `decisions/` 또는 `policies/` 갱신
- 작업 진행 시작하면 → `task-logs/` 로 변환