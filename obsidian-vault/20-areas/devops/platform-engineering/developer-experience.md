---
title: "Developer Experience (DX) — metric / SPACE / DORA"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:34:00+09:00
tags: [devops, platform-engineering, dx]
---

# Developer Experience (DX) — metric / SPACE / DORA

**[[platform-engineering|↑ platform-engineering]]**

---

## 1. DX 가 중요한 이유

```
좋은 DX → 빠른 개발 → product velocity ↑
나쁜 DX → 좌절 / 이직 / 채용 어려움
```

→ Platform team 의 KPI = DX.

---

## 2. DORA 4 metric (★ DevOps 표준)

> "Accelerate" (Forsgren / Humble / Kim)

| | 무엇 | elite |
| --- | --- | --- |
| **Deployment Frequency** | 얼마나 자주 prod 배포 | 하루 여러 번 |
| **Lead Time for Changes** | commit → prod 까지 시간 | < 1시간 |
| **Change Failure Rate** | 배포 중 실패 비율 | 0-15% |
| **Mean Time to Recovery** | 장애 복구 시간 | < 1시간 |

→ 매 quarter 측정 → trend.

---

## 3. SPACE framework

> "SPACE of Developer Productivity" (Forsgren 외)

| | 무엇 |
| --- | --- |
| **S** atisfaction & well-being | 만족도, burnout |
| **P** erformance | 코드 / business outcome |
| **A** ctivity | commit / PR / 회의 |
| **C** ommunication & collaboration | 의사소통 |
| **E** fficiency & flow | 방해 빈도 / context switching |

→ 한 가지 metric 만 X — 여러 측면.

---

## 4. friction 측정

```
- onboard 시간 (신규 입사 → 첫 PR merge)        — 1주 vs 1일
- local dev setup 시간                          — 4h vs 30min
- 새 service 만들기 시간                         — 1일 vs 30min
- CI 실패 → 재시작 비율                          — 10% vs 1%
- "ticket / question" 빈도                       — 50/wk vs 5/wk
- 사용자가 platform 우회하는 빈도                  — 빈번 = signal
```

---

## 5. survey 예 (분기)

```
Q1. platform 도구가 일을 빠르게 해주나요?
   - 1 (어렵게) ~ 5 (매우 빠르게)

Q2. CI 가 느려서 좌절한 횟수 (지난 주)?
Q3. document 가 충분히 명확한가요?
Q4. 가장 답답한 영역?
Q5. 추천 변경?

→ NPS-style: 0-10, "추천 하시겠습니까?"
   - promoter (9-10) - detractor (0-6) = NPS
```

---

## 6. tracking 도구

| | 무엇 |
| --- | --- |
| **DORA Metrics** | LinearB / Sleuth / Faros / 자체 |
| **DX (Devex)** | Pluralsight Flow, Code Climate Velocity |
| **Backstage Tech Insights** | scorecard / metric |
| **Cortex / OpsLevel** | scorecard |
| **GitHub / GitLab** | 기본 metric 노출 |
| **자체** | Athena / BigQuery + dashboard |

---

## 7. fast feedback loops (★)

```
좋은 DX = 짧은 loop:

local:
  save → build → test  < 1 min (hot reload)

PR:
  push → CI → 결과       < 10 min

deploy:
  merge → staging        < 15 min
  manual promote prod    < 5 min

incident:
  alert → on-call ack   < 5 min
```

→ **feedback loop ↑ = DX ↓**.

---

## 8. golden signal of DX

```
1. flow state 유지 시간 / day        — interruption ↓
2. context switch 횟수                — meeting 줄이기
3. tool 학습 시간                     — golden path
4. 새 service 시작 부담               — scaffolding
5. 디버깅 시간 vs 코딩 시간            — observability
6. PR review 대기 시간                — 자동 + culture
```

---

## 9. 개선 예시

| friction | 개선 |
| --- | --- |
| 새 service = 1일 | scaffolding → 30분 |
| local dev 4h | docker-compose dev env / Tilt / Skaffold |
| CI 30분 | cache / parallel / test selection |
| flaky test | retry + quarantine + root cause |
| document 파편 | TechDocs 통합 |
| secrets 관리 | Vault + ESO |
| approval 1주 | self-service + policy |

---

## 10. anti-pattern (DX 망침)

1. **ticket-based ops** — bottleneck.
2. **stale document** — 잘못된 정보가 더 나쁨.
3. **flaky CI** — "retry 하면 됨" 분위기.
4. **tool fragmentation** — 5개 다른 dashboard.
5. **on-call burnout** — 매일 page → 이직.
6. **slow build** — 30분 wait.
7. **너무 많은 meeting** — flow state X.
8. **lack of ownership** — "내 일 아님".

---

## 11. 함정

1. **DORA 만 — overoptimize** — quality 무시.
2. **activity metric (PR/commit) 만** — 진짜 outcome X.
3. **개발자 surveillance** — 신뢰 무너짐.
4. **개선 시도 → metric 조작** — culture 문제.
5. **단발성 survey** — 매 분기 + 비교.

---

## 12. 관련

- [[platform-engineering|↑ platform-engineering]]
- [[idp-concepts]]
- [[../cicd/cicd|↗ CI/CD]]
- [[../sre/toil-reduction|↗ toil reduction]]
