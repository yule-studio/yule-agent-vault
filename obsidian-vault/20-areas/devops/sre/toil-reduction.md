---
title: "Toil reduction — 반복 작업 제거"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:10:00+09:00
tags: [devops, sre, toil, automation]
---

# Toil reduction — 반복 작업 제거

**[[sre|↑ sre]]**

---

## 1. Toil 이란

Google SRE Book 정의:

> 반복적이고, 자동화 가능하고, value 가 없으며, 서비스 성장에 비례해 증가하는 작업.

체크리스트:
- ☐ manual (사람 손)
- ☐ repetitive (반복)
- ☐ automatable (자동화 가능)
- ☐ tactical (대응적, 전략적 X)
- ☐ no enduring value (한 번 하고 흔적 X)
- ☐ scales linearly (서비스 성장에 비례)

→ 3개 이상 해당이면 toil.

---

## 2. 왜 toil 줄여야 하나

### 왜 필요
- 사람 시간 = 가장 비싼 resource.
- toil 이 많으면 새 일 못 함.
- toil 은 사기 저하 (지루함, 야근).

### 안 하면 문제
- 운영자가 burnout.
- 시스템이 사람에 의존 (사람 빠지면 무너짐).
- 회사 성장 = 사람 증원만.

### 대안
- "toil 자체를 할 사람을 더 뽑는다" — scale 안 됨.
- 외주 / 운영 위탁 — 도메인 지식 손실.

### 트레이드오프
- 자동화 = 초기 시간 투자 (코드 / 테스트).
- 잘못된 자동화 = 더 큰 fault.

---

## 3. 50% rule (Google SRE)

> SRE 의 운영 시간이 **50% 를 넘으면 안 된다**.

- 50% 미만 = 자동화 / 개선 (engineering)
- 매주 측정 → toil ratio
- 50% 초과 → product 팀에 작업 차단 / 충원 요청

---

## 4. toil 식별

```
1. 매주 ticket / page / manual task 기록
2. category 별 분류 (deploy, restart, scale, debug, config update, ...)
3. 시간 × 빈도 = total toil
4. high-cost category 부터 자동화 후보
```

---

## 5. 자동화 패턴

| 패턴 | 무엇 | 예 |
| --- | --- | --- |
| **self-service** | UI / CLI 로 제공 | "deploy → CD 가 알아서" |
| **chatops** | Slack 명령 | `/deploy v1.2.3` |
| **auto-remediation** | alert → 자동 처리 | OOM → restart |
| **runbook automation** | runbook 을 코드로 | Ansible / Rundeck |
| **GitOps** | git push → 배포 | ArgoCD / Flux |
| **IaC** | infra 코드화 | [[../iac/iac\|↗ IaC]] |

---

## 6. auto-remediation 예시

```yaml
# k8s HPA — CPU 기반 자동 확장
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: {name: web}
spec:
  scaleTargetRef: {kind: Deployment, name: web}
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource: {name: cpu, target: {type: Utilization, averageUtilization: 70}}
```

→ "트래픽 spike → 사람 호출 X, k8s 가 자동 scale".

---

## 7. chatops 예시

```
Slack:
> /deploy web v1.2.3 to staging
Bot: ✅ Deploying web@v1.2.3 to staging
     → https://argocd.example.com/applications/web

> /scale web to 10 replicas
Bot: ✅ web scaled to 10 (was 5)

> /incident start "checkout 500 errors"
Bot: 🚨 INC-2026-0512-001 created
     → #incident-2026-0512-001 (new channel)
     → on-call paged: @alice (primary), @bob (secondary)
```

---

## 8. runbook → automation 진화

```
Stage 1: manual runbook (md 문서)
Stage 2: scripted runbook (shell / Python)
Stage 3: chatops button ("Run runbook X")
Stage 4: auto-trigger (alert → 자동 실행 + 알림)
Stage 5: 자동 해결 — runbook 없어짐 (root cause 제거)
```

→ **5단계가 최종 목표**.

---

## 9. KPI

| 지표 | 목표 |
| --- | --- |
| toil % | < 50% |
| MTTR (Mean Time To Recovery) | ↓ |
| MTTD (Mean Time To Detect) | ↓ |
| deploy frequency | ↑ |
| change failure rate | ↓ |
| recurring incident count | 0 |

→ DORA metrics 와 연결.

---

## 10. 함정

1. **자동화에 자동화** — 사람보다 자동화가 더 복잡 → maintain 어려움.
2. **잘못된 self-service** — 위험한 명령을 누구나 (예: `/db drop-table`).
3. **silent failure** — 자동화 실패 시 알람 없음.
4. **escape hatch 없음** — 자동화 끌 수 없음.
5. **자동화 = 한 번 하고 끝** — drift / 변화 추적 안 함.

---

## 11. 관련

- [[sre|↑ sre]]
- [[runbook]]
- [[../iac/iac]]
- [[../cicd/cicd]]
