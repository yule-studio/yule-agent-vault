---
title: "unsorted — 분류 미정"
kind: knowledge
project: agent-inbox
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T18:30:00+09:00
tags:
  - inbox
  - unsorted
  - draft
---

# unsorted — 분류 미정

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 10 노트 |

**[[../inbox|↑ 00-inbox/]]**

## 이 폴더가 묶는 것

운영 중 발견한 **관찰 / 후속 작업 후보 / 메타 메모** 중 아직 결정 /
작업 / 정책 으로 변환되지 않은 항목. 분류가 결정되면 적절한 폴더로
이주.

## 현재 노트 (10)

### 관찰 (observation)

- [[observation-bot-noise-pattern]] — Discord 봇 다중 응답 noise
- [[observation-vault-graph-isolated-nodes]] — 그래프 외톨이는 broken backlink stub
- [[observation-anthropic-sandbox-discord-block]] — sandbox 의 outbound 차단

### 후속 작업 (followup)

- [[followup-f15-governance-test]] — F15 governance 회귀 테스트
- [[followup-hr-finance-skeletons]] — HR/Finance/Sales-CS/Legal 부서 골격
- [[followup-discord-webhook-secret-setup]] — Discord CI webhook secret
- [[followup-pm-skills-catalog]] — PM Skills catalog 도입
- [[followup-engineering-knowledge-collector]] — 부서 자동 자료 수집기 검증

### 메타 (meta)

- [[meta-commit-splitting-effect]] — commit 분리 정책의 효과
- [[meta-100-percent-operational-framework]] — 100% 운영 틀의 정의

## 운영 규칙

1. **임시 영역**. 분류 결정되면 즉시 이주.
2. 후속 작업 (followup) 이 사이클 시작 시점에 작업 로그로 변환.
3. 관찰 (observation) 이 정책 영향 있으면 `decisions/` 노트로 변환.
4. 메타 노트는 hub (`_moc/`) 로 흡수 또는 vault root README 에 반영.
