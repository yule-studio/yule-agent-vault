---
title: "SRE — Site Reliability Engineering ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:05:00+09:00
tags: [area, devops, sre]
---

# SRE — Site Reliability Engineering ★

**[[../devops|↑ devops]]**

---

## 1. SRE 란

Google 이 정의한 운영 모델:

> "SRE is what happens when you ask a software engineer to design an operations team."  
> — Ben Treynor (Google VP)

- **Dev 와 Ops 의 통합** (DevOps 의 한 구현체).
- 운영의 "toil" (반복 작업) 을 코드로 제거.
- 신뢰성 (reliability) 을 정량적으로 관리 (SLO).

---

## 2. SRE 핵심 원칙 (Google SRE Book)

1. **에러 예산 (Error Budget)** — 100% reliability ≠ 목표.
2. **toil 최소화** — 운영 시간의 50% 이상이 코딩이어야.
3. **monitoring 4 Golden Signals** — latency, traffic, errors, saturation.
4. **자동화** — 사람 손 X (auto-remediation).
5. **postmortem (blameless)** — 사람 비난 X, system 개선.
6. **simplicity** — 복잡도가 곧 fragility.
7. **release engineering** — canary, rollback 가능.
8. **capacity planning** — load test + forecast.

---

## 3. SRE vs DevOps

| | DevOps | SRE |
| --- | --- | --- |
| 정의 | 문화 / 철학 | Google 의 구현 모델 |
| 책임 | dev + ops 협력 | dev + reliability 정량 관리 |
| KPI | deploy 빈도 | SLO compliance |
| 조직 | 팀 구조 다양 | SRE 팀 + product 팀 |

→ "DevOps 는 What, SRE 는 How."

---

## 4. 하위 영역

- [[slo-management]] — SLI/SLO/Error Budget 운영
- [[toil-reduction]] — 운영 자동화
- [[on-call]] — 호출 rotation / handoff
- [[incident-response]] — 발생 대응 흐름
- [[postmortem]] — blameless postmortem 작성
- [[chaos-engineering]] — Chaos Mesh / LitmusChaos
- [[runbook]] — runbook 작성 표준
- [[disaster-recovery]] — DR / RTO / RPO

---

## 5. 학습 순서

1. Day 1 — SLO 개념 ([[slo-management]] + [[../monitoring/slos-and-sli|monitoring SLO]])
2. Day 2 — alerting + on-call ([[on-call]] + [[../monitoring/alerting|monitoring alerting]])
3. Day 3 — incident response + postmortem ([[incident-response]], [[postmortem]])
4. Day 4 — toil 감소 ([[toil-reduction]])
5. Day 5+ — chaos engineering, DR ([[chaos-engineering]], [[disaster-recovery]])

---

## 6. 추천 도서

- **Google SRE Book** (무료) — sre.google/books/
- **SRE Workbook** — 실전 (무료)
- **Seeking SRE** — 다양한 회사 사례
- **Implementing Service Level Objectives** (Alex Hidalgo)

---

## 7. 관련

- [[../devops|↑ devops]]
- [[../monitoring/monitoring|↗ monitoring]]
- [[../kubernetes/kubernetes|↗ kubernetes]]
