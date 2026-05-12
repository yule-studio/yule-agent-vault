---
title: "Discord CI webhook secret 셋업 가이드"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: draft
created_at: 2026-05-12T18:30:00+09:00
tags:
  - unsorted
  - inbox
---

# Discord CI webhook secret 셋업 가이드

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[unsorted|↑ unsorted/]]** · **[[../inbox|↑ 00-inbox/]]**

## 잔여 작업

`DISCORD_CI_WEBHOOK_URL` secret 이 GitHub repo 에 미설정. PR 빌드 알림이 동작하지 않음.

## 범위

`.github/workflows/ci.yml` 의 notify job 이 secret 사용. 사용자 호스트에서 `gh secret set DISCORD_CI_WEBHOOK_URL` 실행 필요.

## 우선순위

다음 PR 직전. secret 셋업 후 fail / success 알림이 Discord 에 도착.

## 관련

`vars.DISCORD_CI_FAIL_IMAGE_URL` / `vars.DISCORD_CI_SUCCESS_IMAGE_URL` 은 이미 설정됨.

## 분류 후 이주 후보

- 정책 영향 있으면 → `decisions/` 또는 `policies/` 갱신
- 작업 진행 시작하면 → `task-logs/` 로 변환