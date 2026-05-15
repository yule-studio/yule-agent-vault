---
title: "Runbook — 표준 절차서"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:22:00+09:00
tags: [devops, sre, runbook]
---

# Runbook — 표준 절차서

**[[sre|↑ sre]]**

---

## 1. runbook 이란

특정 alert / scenario 에 대한 **단계별 대응 절차**.

- 새벽 3시에 자고 일어난 on-call 이 보고 따라 할 수 있어야.
- alert annotation 에서 link 로 연결.

---

## 2. 왜 필요

### 왜 필요
- 사람마다 다른 대응 → 비일관성.
- 신참 on-call 도 즉시 대응 가능.
- tribal knowledge → 명시 지식.

### 안 하면 문제
- "Senior 한테 물어봐" — 야간에 깨움.
- 같은 alert 마다 디버깅 처음부터.
- 핵심 인력 의존도 ↑.

### 대안
- "다 외워" — 비현실적.
- 위키 검색 — 너무 많아서 못 찾음.

### 트레이드오프
- runbook 작성 시간.
- 오래되면 stale (deploy 변경 후 runbook X).

---

## 3. 표준 템플릿

```markdown
# Runbook — HighErrorRate (web-api)

## TL;DR
web-api 의 5xx 에러율 > 5% (5분 평균).
→ 보통 deploy 또는 downstream 문제.
→ rollback 또는 downstream check.

## Triggers
- AlertManager: `HighErrorRate` (severity: page)
- 또는 customer 보고

## Impact 평가 (2 min)
1. Grafana → "Web API Overview" 열기
2. error rate, RPS, p95 latency 확인
3. 어떤 endpoint? user 영향 범위?

## 조사 절차

### Step 1 — 최근 deploy 확인
```bash
kubectl get deployment web-api -n prod -o wide
argocd app history web-api
```

### Step 2 — log 확인
```bash
kubectl logs -n prod -l app=web-api --tail=100 | grep ERROR
# 또는 Loki: {app="web-api"} |= "ERROR"
```

### Step 3 — downstream 확인
- DB connection: HikariCP saturation
- Redis: timeout?
- 외부 API: 5xx?

## 대응

### Option A — rollback (가장 빠름)
```bash
argocd app rollback web-api <prev-revision>
# 또는
kubectl rollout undo deployment/web-api -n prod
```

### Option B — scale up (resource 부족 시)
```bash
kubectl scale deployment/web-api -n prod --replicas=20
```

### Option C — feature flag off
```bash
curl -X POST $FLAGSMITH/api/v1/flags/checkout-v2 \
     -H "Authorization: $TOKEN" \
     -d '{"enabled": false}'
```

## 검증
- error rate < 1% 5분 sustained?
- p95 latency 정상?

## Escalation
- 10분 안에 mitigate X → IC + Tech Lead 호출
- DB / Redis 문제 → DBA / platform 팀

## Related
- [[incident-response]]
- Grafana dashboard: ...
- previous incident: INC-2026-0512-001
```

---

## 4. alert ↔ runbook link

```yaml
- alert: HighErrorRate
  expr: ...
  annotations:
    summary: "web-api 5xx > 5%"
    runbook_url: "https://runbooks.example.com/web-api/high-error-rate"
```

→ AlertManager / PagerDuty 에서 클릭 시 즉시 runbook 으로.

---

## 5. runbook 종류

| Type | 예 |
| --- | --- |
| **alert response** | "HighErrorRate" alert 처리 |
| **operational** | "DB failover" / "scale up" |
| **disaster recovery** | "region failover" |
| **maintenance** | "정기 점검 절차" |
| **onboarding** | "신규 on-call 첫 주" |

---

## 6. 작성 원칙

```
1. 명령은 copy-paste 가능 (full 명령, placeholder ${VAR} 명시)
2. 출력 예시 포함 (정상 vs 비정상)
3. 단계 별 검증 ("성공 시 ... 확인")
4. abort 조건 ("X 가 아니면 escalate")
5. 마지막 update 일자 + 작성자
6. 매 분기 검증 (drill)
```

---

## 7. runbook 자동화 진화

```
Stage 1: markdown (현재 표준)
Stage 2: 명령 portion 을 shell script
Stage 3: chatops button ("/runbook web-api/high-error-rate")
Stage 4: alert → 자동 실행 (auto-remediation)
Stage 5: root cause 제거 → runbook 자체 불필요
```

---

## 8. 함정

1. **runbook 부재** — 새벽 3시 깜깜이.
2. **stale runbook** — 명령이 더 이상 동작 X.
3. **추상적 step** — "log 확인" — 어느 log? 어떤 패턴?
4. **escalation 조건 없음** — 무한 시도.
5. **link 없음** — 알고 있어야 사용 가능 → 못 찾음.
6. **검증 없음** — 실제 fail 시 동작 안 함.

---

## 9. drill (정기 검증)

```
quarterly: 모든 P0/P1 runbook 을 실제 실행 (staging)
  - 명령 동작?
  - 시간 < target?
  - 명령 출력이 문서와 일치?
  - 갱신 필요?
```

→ chaos engineering 과 연결.

---

## 10. 관련

- [[sre|↑ sre]]
- [[incident-response]]
- [[on-call]]
- [[toil-reduction]]
