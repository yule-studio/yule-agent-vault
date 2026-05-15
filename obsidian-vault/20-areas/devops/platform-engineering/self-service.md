---
title: "Self-service — UI / CLI / GitOps"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:32:00+09:00
tags: [devops, platform-engineering, self-service]
---

# Self-service — UI / CLI / GitOps

**[[platform-engineering|↑ platform-engineering]]**

---

## 1. 왜

```
ticket 기반:
  개발자: "RDS 만들어 주세요"
  ops: "어떤 size? ..."
  ↳ 3일 wait
  ↳ ops bottleneck

self-service:
  개발자: idp create db --size small
  ↳ 5분 안에 ready
```

→ ticket = scale 안 됨.

---

## 2. self-service surface

```
[UI] Backstage / Port           ← non-technical (PM 등)
 ↑
[CLI] internal tool             ← 일반 dev
 ↑
[GitOps] PR to infra repo       ← 표준, audit-friendly
 ↑
[IaC code] Terraform / Pulumi   ← infra team / platform team
```

→ 같은 정책을 여러 surface 로 제공.

---

## 3. UI (★ Backstage)

```
- 신규 service create
- DB / cache 발급
- pipeline trigger
- 환경 변수 / secret rotation
- on-call rotation 등록
- scorecard 확인
- TechDocs 검색
```

→ "click → ready". non-technical OK.

---

## 4. CLI

```bash
# 예 (회사 자체 도구)
idp service create my-app --type spring-boot --team platform
idp service list
idp service describe my-app
idp service deploy my-app --env staging --tag v1.2.3
idp service rollback my-app --env prod

idp db create --kind postgres --size small --owner my-app
idp secret set my-app/db-password "value"
idp pipeline status my-app
idp logs my-app -f
```

→ 개발자 친화. CI/CD 통합.

---

## 5. GitOps (★ 표준)

```
infra-as-data:
  infra-repo/
    services/
      my-app/
        deployment.yaml
        service.yaml
        ingress.yaml
    databases/
      my-app-db/
        postgres.yaml   ← Crossplane / Terraform yaml

→ PR + review + merge → ArgoCD apply
```

장점:
- 모든 변경 history
- code review
- rollback (git revert)
- audit (compliance)
- multi-env (branch / overlay)

---

## 6. ChatOps (Slack)

```
@platform-bot deploy my-app v1.2.3 to staging
→ ✅ deploying...
→ ✅ deployed

@platform-bot incident start "checkout 5xx"
→ 🚨 INC-001 created
→ #inc-001 channel
→ pager @alice

@platform-bot scale my-app --replicas 10
→ ⚠️  scaling above policy. approval required.
@manager approve
→ ✅ scaled
```

→ 친숙한 채널 + audit log.

---

## 7. policy (★ guardrails)

self-service 라도 가드 필요:

```yaml
# OPA / Conftest 정책 예
deny[msg] {
    input.kind == "Postgres"
    input.spec.size == "huge"
    input.spec.owner != "platform"
    msg := "huge size 는 platform team 만 허용"
}

deny[msg] {
    input.kind == "Service"
    input.spec.replicas > 10
    not input.metadata.annotations["approved-by"]
    msg := "10 replica 이상은 approval 필요"
}
```

→ Crossplane / Terraform CI 에서 검증.

---

## 8. approval flow

```
변경 종류                        | 흐름
-------------------------------|------
새 service                      | self-service (template)
기존 service deploy             | self-service
config 변경                     | PR review (peer)
size up                         | PR review
DB schema migration             | PR + DBA review
new database                    | self-service (size 제한)
production credentials          | request + approval (JIT)
IAM policy 변경                 | PR + security review
DNS 변경                        | PR + ops review
```

→ blast radius 따라 차등.

---

## 9. observability of self-service

```
- 모든 self-service 액션 audit log
- "Top 10 actions per week"
- 사용자별 활동
- failure rate (실패 = friction)
- median time-to-complete

→ platform 개선의 신호.
```

---

## 10. 함정

1. **self-service 가 너무 광범위** — 사고. policy + guardrail.
2. **guardrail 너무 빡빡** — 우회 시도.
3. **UI / CLI / GitOps 정책 불일치** — 한 곳만 enforce.
4. **failure mode 없음** — error 시 사용자 어찌할지 모름.
5. **audit log 없음** — 누가 했나 모름.
6. **non-technical 무시** — PM / designer 가 못 씀.

---

## 11. 관련

- [[platform-engineering|↑ platform-engineering]]
- [[backstage]]
- [[scaffolding]]
- [[../sre/toil-reduction|↗ toil reduction]]
