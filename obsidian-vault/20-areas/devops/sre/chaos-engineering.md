---
title: "Chaos Engineering"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:25:00+09:00
tags: [devops, sre, chaos]
---

# Chaos Engineering

**[[sre|↑ sre]]**

---

## 1. 무엇

> "프로덕션 시스템의 신뢰도를 측정하기 위해, 의도적으로 결함을 주입하는 실험."  
> — Netflix Chaos Monkey

- 시스템이 **fail 에도 견디는지** 확인 (resilience).
- 사실은 항상 일어남 — 차라리 통제된 환경에서.

---

## 2. 왜 필요

### 왜 필요
- "장애에도 동작" 가정이 검증 안 됨.
- 한 component 실패 → 전체 cascade 가 실제 일어남.
- runbook / DR 검증.

### 안 하면 문제
- 진짜 incident 때 "이게 됐어야 하는데..." 발견.
- multi-region failover 안 됨 (검증 안 했음).

### 대안
- "그냥 모니터링 잘하자" — 미검증 가정 그대로.
- staging 에서만 — prod 환경 차이 무시.

### 트레이드오프
- 실제 사용자 영향 (game day 시).
- 잘못 설계 시 진짜 incident.

---

## 3. 원칙 (Netflix Principles)

1. **steady state 정의** — 정상 metric (RPS, error rate, latency).
2. **hypothesis** — "X 가 fail 해도 steady state 유지".
3. **실제와 같은 환경** — 가능하면 prod.
4. **계속 자동** — CI 처럼.
5. **blast radius 최소화** — 1% traffic 부터.

---

## 4. 도구

| 도구 | 무엇 |
| --- | --- |
| **Chaos Monkey** | Netflix, 무작위 인스턴스 kill (legacy) |
| **Chaos Mesh** | k8s 네이티브, CNCF |
| **LitmusChaos** | k8s, GitOps 친화 |
| **Gremlin** | SaaS, UI 친화 |
| **AWS Fault Injection** | AWS 통합 |
| **Pumba** | Docker 대상 |
| **toxiproxy** | 네트워크 결함 주입 (latency / packet loss) |

---

## 5. Chaos Mesh 설치 + 실행

```bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh \
    -n chaos-mesh --create-namespace
```

### Pod kill

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: kill-web-pod
  namespace: chaos-mesh
spec:
  action: pod-kill
  mode: one                  # 1개만
  selector:
    namespaces: [prod]
    labelSelectors: {app: web}
  scheduler:
    cron: "@every 1h"        # 1시간마다
```

### Network latency

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata: {name: redis-latency}
spec:
  action: delay
  mode: all
  selector:
    namespaces: [prod]
    labelSelectors: {app: redis}
  delay:
    latency: "200ms"
    jitter: "50ms"
  duration: "10m"
```

### CPU stress

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata: {name: cpu-stress}
spec:
  mode: one
  selector:
    namespaces: [prod]
    labelSelectors: {app: web}
  stressors:
    cpu: {workers: 4, load: 80}
  duration: "5m"
```

---

## 6. 실험 시나리오

| 시나리오 | 가설 | 검증 |
| --- | --- | --- |
| Pod kill | k8s 가 자동 reschedule, traffic 영향 X | RPS / error rate |
| AZ down | multi-AZ failover, latency 약간 ↑ | availability |
| Region down | DR region 으로 failover | RTO / RPO |
| Redis down | cache miss → DB load ↑, 동작 유지 | error rate, DB CPU |
| DB master down | failover (5-30s) | downtime |
| Network partition | circuit breaker open, fallback | error rate |

---

## 7. Game Day

```
- 정기 (분기) 운영
- 전 팀 참여 (eng + product + comms)
- 시나리오 미리 비공개
- IC / Tech Lead / Scribe 역할
- postmortem 처럼 결과 정리
- action item → 다음 Game Day 까지 fix
```

---

## 8. 단계별 진입

```
Stage 1: dev 환경 — pod kill (안전)
Stage 2: staging — network / CPU
Stage 3: prod (1% traffic, off-peak)
Stage 4: prod (peak time, full)
Stage 5: 자동 (CI 와 같은 빈도)
```

→ **갑자기 prod 에서 시작 X**.

---

## 9. 안전 장치

- **abort button** — Chaos Mesh / SaaS 의 즉시 중단
- **blast radius** — 1 pod / 1 zone / 1% traffic 부터
- **시간 제한** — 5min / 10min
- **off-peak** — 새벽 점검 시간
- **monitoring on** — 실시간 dashboard 보면서

---

## 10. 함정

1. **prod 에서 처음** — 진짜 incident 됨.
2. **abort 없음** — 통제 불가.
3. **monitoring 안 봄** — 영향 모르고 진행.
4. **runbook 없음** — fail 시 어떻게 대응?
5. **너무 좁은 범위** — 항상 Pod kill 만 → 한 가정만 검증.
6. **postmortem 없음** — Game Day 결과 학습 0.

---

## 11. 관련

- [[sre|↑ sre]]
- [[disaster-recovery]]
- [[runbook]]
- [[../monitoring/monitoring|↗ monitoring]]
