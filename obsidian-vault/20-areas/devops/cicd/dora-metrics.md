---
title: "DORA metrics — DevOps 의 4 KPI"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:28:00+09:00
tags: [devops, cicd, dora, metric]
---

# DORA metrics — DevOps 의 4 KPI

**[[cicd|↑ cicd]]**

---

## 1. 4 metric (Accelerate, Forsgren et al.)

```
1. Deployment Frequency       — 얼마나 자주 deploy
2. Lead Time for Changes      — commit → prod 시간
3. Change Failure Rate (CFR)  — deploy 중 실패 %
4. Mean Time to Recovery     — fail 후 복구 시간
```

→ Google 의 DORA report = 매년 측정 발표.

---

## 2. elite vs low 기준

```
Elite:
  Deploy Frequency:    여러 번 / day
  Lead Time:           < 1 hour
  CFR:                 0-15%
  MTTR:                < 1 hour

High:
  Deploy Frequency:    weekly - monthly
  Lead Time:           < 1 day
  CFR:                 16-30%
  MTTR:                < 1 day

Medium:
  Deploy Frequency:    monthly - 6m
  Lead Time:           1-6 month
  CFR:                 16-30%
  MTTR:                1-7 day

Low:
  Deploy Frequency:    < 6m
  Lead Time:           > 6m
  CFR:                 16-30%
  MTTR:                > 6m
```

---

## 3. 왜 중요

```
연구 결과:
  Elite vs Low 의 회사:
    - 매출 성장 2.5x
    - throughput 1.6x
    - 직원 만족도 ↑

→ DORA = velocity + stability 의 균형.
   둘 다 좋아짐. trade-off X (Accelerate 의 핵심 메시지).
```

---

## 4. Deployment Frequency

```
측정:
  prod 배포 횟수 / period
  
high:
  매일 N번
  feature flag 로 release 분리

improvement:
  - trunk-based development
  - CI / CD 자동화
  - small batch
  - test 자동화
```

---

## 5. Lead Time

```
측정:
  git commit → prod deploy 시간
  
일부 정의:
  - first commit ~ deploy
  - PR open ~ deploy
  - PR merge ~ deploy

high:
  < 1 hour

improvement:
  - CI 빠름
  - manual gate 줄임
  - 작은 PR
  - automated test
```

---

## 6. Change Failure Rate (CFR)

```
측정:
  rollback / hotfix / 실패 deploy / 비스가 깨짐
  / 전체 deploy 수

low (좋음):
  0-15%

너무 낮음 (이상):
  - test 가 prod-like 아님
  - 또는 너무 느린 deploy

improvement:
  - canary / progressive
  - automated test
  - feature flag
  - observability
```

---

## 7. MTTR

```
측정:
  incident 시작 ~ 복구 시간
  
low:
  < 1 hour

improvement:
  - rollback 자동
  - feature flag off
  - 모니터링 / alert 빠른 감지
  - runbook
  - on-call rotation
```

---

## 8. 측정 (★)

```
git log + deployment record + incident:

deploy:
  - GitHub release / tag
  - ArgoCD sync event
  - CD pipeline 의 prod step 시각

failure:
  - issue tracker 의 "incident" label
  - rollback commit
  - PagerDuty alert

도구:
  - LinearB ★
  - Sleuth
  - Haystack
  - GitHub Advanced Security
  - Datadog DevOps Lifecycle
  - Faros AI
  - Code Climate Velocity
  - 자체 (BigQuery + DAG)
```

---

## 9. 자체 측정 query 예

```sql
-- Deploy Frequency (지난 30일)
SELECT
    DATE_TRUNC('day', deployed_at) as day,
    COUNT(*) as deploys
FROM deployments
WHERE env = 'prod' AND deployed_at >= NOW() - INTERVAL '30 days'
GROUP BY 1
ORDER BY 1;

-- Lead Time
SELECT
    AVG(EXTRACT(EPOCH FROM (deployed_at - first_commit_at)) / 60) as avg_lead_time_min,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (deployed_at - first_commit_at)) / 60) as p50_min,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (deployed_at - first_commit_at)) / 60) as p95_min
FROM deployments d
JOIN commits c ON c.id = d.first_commit_id
WHERE d.env = 'prod';

-- CFR
SELECT
    COUNT(*) FILTER (WHERE failed) * 100.0 / COUNT(*) as cfr_pct
FROM deployments
WHERE env = 'prod' AND deployed_at >= NOW() - INTERVAL '30 days';

-- MTTR
SELECT
    AVG(EXTRACT(EPOCH FROM (resolved_at - started_at)) / 60) as avg_mttr_min
FROM incidents
WHERE severity IN ('P0', 'P1')
  AND started_at >= NOW() - INTERVAL '90 days';
```

---

## 10. 개선 lever (★)

### Deploy Frequency 늘리기

```
☐ trunk-based development (long-lived branch 없음)
☐ feature flag (deploy ≠ release)
☐ CI 빠름 (< 10분)
☐ test 빠름 + parallel
☐ deploy 자동 (manual gate 줄임)
☐ small PR (review 빠름)
```

### Lead Time 줄이기

```
☐ CI build / test 시간 절감
☐ PR review SLA (1 day)
☐ rebase / squash 정책
☐ deploy 자동
☐ pipeline 단계 줄임
```

### CFR 줄이기

```
☐ test coverage 의미 있게
☐ canary / progressive
☐ smoke test in prod
☐ DB migration tool (Liquibase / Flyway)
☐ schema compatibility
☐ chaos engineering
```

### MTTR 줄이기

```
☐ automated rollback
☐ feature flag (즉시 off)
☐ alert tuning (false positive ↓)
☐ runbook + 자동화
☐ observability (찾기 빠름)
☐ on-call training
```

---

## 11. 5번째 metric — Reliability (★ 2021+)

```
Operational Performance 추가:
  - SLO compliance
  - 사용자 영향 incident 수

Stability + Throughput + Reliability.
```

---

## 12. anti-pattern

```
❌ 한 metric 만 optimize
  → deploy 빈도 ↑ but CFR 폭주
  → balance.

❌ team 비교
  → 정치적. 개선 lever 와 함께.

❌ metric 의 manipulation
  → 작은 deploy 분리, fake CFR
  → trust 무너짐.

❌ metric 만 자동화, 개선 안 함
  → dashboard 만.
```

---

## 13. 함정

1. **수동 측정** — 시간 낭비, 부정확.
2. **production 만** — pipeline 전체.
3. **average 만** — p50/p95/p99 도.
4. **DORA 가 KPI 의 전부** — quality / DX 도.
5. **개선 lever 없음** — metric 만 보고.
6. **team 별 비교** — context 다름.

---

## 14. 관련

- [[cicd|↑ cicd]]
- [[../platform-engineering/developer-experience|↑ DX]]
- [[../sre/sre|↑ SRE]]
- [[pipeline-patterns]]
