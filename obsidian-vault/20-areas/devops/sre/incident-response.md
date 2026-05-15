---
title: "Incident response — 발생 대응"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:16:00+09:00
tags: [devops, sre, incident]
---

# Incident response — 발생 대응

**[[sre|↑ sre]]**

---

## 1. incident 란

- "정상 상태에서의 의도되지 않은 이탈".
- severity 별 분류 (P0/P1/P2/P3).
- 발견 → 대응 → 회복 → 학습 흐름.

---

## 2. severity 표준

| | 의미 | 예 | 대응 |
| --- | --- | --- | --- |
| **P0** | 전사 서비스 down | 전체 site 500 / 결제 불가 | 즉시 전원 호출, war room |
| **P1** | major 기능 down | 검색 불가 / login 일부 fail | on-call + 1명 backup |
| **P2** | minor 기능 / degraded | latency 2x, 일부 사용자 | on-call 단독 |
| **P3** | 자동 복구 / non-urgent | disk warning | ticket only |

---

## 3. 대응 흐름 (4 phase)

```
1. DETECT (감지)
   - alert / customer 보고 / dashboard

2. RESPOND (대응)
   - acknowledge (5min)
   - severity 평가
   - war room 소집 (P0/P1)
   - 역할 분담 (IC / scribe / liaison)

3. RECOVER (회복)
   - 일단 영향 차단 (mitigate) — root cause 는 나중
   - rollback / feature flag off / scale up / restart
   - 사용자 영향 stop

4. LEARN (학습)
   - postmortem (24-48h 안에)
   - action item → 다음 cycle 에 fix
```

---

## 4. 역할 (★ 큰 incident 시)

| 역할 | 책임 |
| --- | --- |
| **IC** (Incident Commander) | 의사결정자. 기술적 작업 X, coord 만. |
| **Tech Lead** | 실제 디버깅 / fix |
| **Communications** (Liaison) | 사내 stakeholder / 고객 / 외부 |
| **Scribe** | timeline 기록 (postmortem 용) |

→ P0 시 4명. P1 = IC + Tech Lead. P2 = on-call 1명.

---

## 5. communication 템플릿

### 5-1. status page 업데이트

```
🟡 Investigating — 13:42 UTC
We are investigating reports of checkout failures.

🟠 Identified — 13:55 UTC
A recent deploy is suspected. Rolling back.

🟢 Resolved — 14:10 UTC
Service is restored. Postmortem to follow.
```

### 5-2. internal Slack

```
🚨 P1 Incident — checkout 500 errors
Channel: #inc-2026-0515-001
IC: @alice
Tech Lead: @bob
Started: 13:42 UTC
Impact: ~30% of checkout requests failing
Current action: investigating recent deploys
```

---

## 6. mitigate 우선 — 자주 쓰는 leverage

```
1. rollback (가장 빠름)
   kubectl rollout undo deployment/web
   argocd app sync web --revision PREVIOUS

2. feature flag off
   curl -X POST flagsmith/api/v1/flags/checkout-v2 \
        -d '{"enabled": false}'

3. scale up
   kubectl scale deployment/web --replicas=20

4. circuit break
   downstream timeout 단축 / disable

5. traffic shift
   Istio VirtualService / weighted route

6. restart
   kubectl delete pod -l app=web
```

→ **root cause 파악은 나중**. 영향 stop 이 우선.

---

## 7. war room 진행

```
13:42 IC: "P1 incident declared. Tech lead?"
13:42 Bob: "I'll lead tech. Looking at deploys."
13:43 Alice (IC): "Comms?"
13:43 Carol: "I've got it. Status page yellow."
13:43 Alice (IC): "Scribe?"
13:43 Dave: "Logging in #inc channel."

13:48 Bob: "Found it — deploy 14:30 added bad query."
13:48 Alice: "Mitigation? Rollback or fix forward?"
13:49 Bob: "Rollback safer. 2 min."
13:49 Alice: "Approved. Carol, prep status update."

13:51 Bob: "Rollback complete. Watching metrics."
13:55 Bob: "Error rate back to baseline."
13:55 Alice: "Carol, status green. Dave, schedule postmortem 24h."
```

---

## 8. severity escalation

```
P2 → P1: 영향이 확장 (1 region → 전체) → IC + Tech Lead 호출
P1 → P0: 결제 / login 등 critical 영향 → 경영진 / Comms 강화
P0 → executive: 외부 노출 / 대량 매출 손실 → CEO 브리프
```

---

## 9. mid-incident 하면 안 되는 것

1. ❌ root cause 디버깅에 너무 매몰 → 사용자 영향 지속
2. ❌ blame ("누가 그 deploy 했어?") → war room 분위기 망침
3. ❌ slack 채널 통일 안 됨 → 정보 분산
4. ❌ scribe 없음 → postmortem 시 timeline 불명
5. ❌ 너무 많은 사람 — 7명 이상이면 cognitive overload

---

## 10. 함정

1. **alert 무시** — 누가 보겠지 → SLA 위반.
2. **혼자 해결 시도** — escalation 안 함 → 야간에 burnout.
3. **rollback 못 함** — DB schema 호환성 / non-reversible.
4. **status page 늦음** — 고객이 먼저 알게 됨 → 신뢰 X.
5. **postmortem skip** — 같은 incident 반복.

---

## 11. 관련

- [[sre|↑ sre]]
- [[postmortem]]
- [[runbook]]
- [[on-call]]
