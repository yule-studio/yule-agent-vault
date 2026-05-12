---
title: "GitHub PR AI 코드 리뷰 봇"
kind: knowledge
project: project-ideas
agent: engineering-agent/tech-lead
status: idea
created_at: 2026-05-12T23:00:00+09:00
tags:
  - project-idea
  - monetization
  - solo-founder
  - saas-ai-code-review-bot
---

# GitHub PR AI 코드 리뷰 봇

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[project-ideas|↑ project-ideas/]]**

## 개요

PR 열면 봇이 자동 리뷰. 보안 / 성능 / 안티패턴. 팀 / 개인 모두 타겟.

## 수익 모델

Pro $19-49/repo/mo. enterprise per-seat.

## 실제 레퍼런스

CodeRabbit (~ARR $5M+ in 1년) / Sweep / Greptile / Korbit. PR-Agent (open-source by Qodo).

## 기술 스택

GitHub Apps + LLM (Claude / GPT) + git diff parser. webhook 처리.

## 진입 장벽

중간 (LLM 비용 / latency). 차별화 = 한국어 리뷰 / 한국 기업 코드 스타일.

## 한국 시장 변형

한국 기업 (네이버 / 카카오 / 토스) 코드 컨벤션 학습. 한국어 PR 코멘트.

## 학습 자료

https://github.com/Codium-ai/pr-agent (open-source) / CodeRabbit 의 marketing playbook
