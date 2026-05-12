---
title: "MOC — Discord ↔ CI 알림 전략"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T15:30:00+09:00
tags:
  - moc
  - hub
  - discord
  - ci
related:
  - _moc.md
---

# MOC — Discord ↔ CI 알림 전략

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 hub |

## 이 hub 가 묶는 것

Discord 채널 ↔ GitHub Actions CI 알림의 경계 / 책임 / 실패 시 동작
정책. CD (배포) 와의 책임 분리 포함.

**[[_moc|↑ MOC 인덱스]]**

## 결정 / 리서치 / 작업 로그

- [[../decisions/decision-discord-ci-and-cd-boundary]] — CI 알림 vs CD
  실행 책임 분리 결정
- [[../research/research-discord-ci-strategy]] — Discord webhook /
  embed / mention 전략 비교
- [[../task-logs/task-log-discord-ci-design]] — 실제 알림 디자인 작업

## 운영 상태

- `.github/workflows/ci.yml` notify job 이 `secrets.DISCORD_CI_WEBHOOK_URL`
  + `vars.DISCORD_CI_FAIL_IMAGE_URL` / `vars.DISCORD_CI_SUCCESS_IMAGE_URL`
  사용
- 사용자 호스트에서 secret 셋업 필요 (sandbox 에서는 차단)

## 관련 hub

- [[issue-73-tech-lead-runtime]] — runtime loop 안의 Discord 게이트웨이
