---
title: "Platform Engineering — IDP / golden path / Backstage"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:22:00+09:00
tags: [area, devops, platform-engineering]
---

# Platform Engineering — IDP / golden path / Backstage

**[[../devops|↑ devops]]**

---

## 1. 무엇

> 개발자가 더 빠르고 안전하게 서비스를 만들 수 있도록  
> 내부에 **product 같은 platform** 을 제공하는 engineering 분야.

→ DevOps 의 진화. "you build it, you run it" 의 부담을 줄임.

---

## 2. 왜 필요

### 왜 필요
- DevOps 부담이 개발자에게 과중 — 학습 / cognitive load.
- 매 service 마다 CI/CD / IaC / monitoring / security 반복 구축.
- 표준 부재 → 운영 복잡도 ↑.

### 안 하면 문제
- 개발자가 80% 시간 = boilerplate / 운영.
- 도구 / 패턴 fragmentation — onboard 비용 ↑.
- security / compliance 일관성 X.

### 대안
- 중앙 DevOps 팀 — bottleneck.
- 각 팀 자체 운영 — duplication.

### 트레이드오프
- platform 자체 운영 부담.
- "product mindset" — 사용자 (개발자) 우선.

---

## 3. IDP (Internal Developer Platform)

```
[개발자]
    ↓ (단순 UI / CLI / git)
[Platform]
    ↓ 추상화
[infra (k8s / cloud)]
```

핵심 기능:
- service scaffolding (template)
- 환경 provisioning (dev / stage / prod)
- CI/CD 자동
- secrets / config
- monitoring / logging 자동 적용
- access (RBAC, secrets)
- documentation 통합

→ "platform-as-a-product".

---

## 4. golden path

```
"the easiest path is also the best path"

새 service 만들 때:
  1. `idp create service my-app --type spring-boot`
  2. 자동으로:
     - git repo 생성 + template
     - CI/CD pipeline 설정
     - dev/stage cluster namespace
     - monitoring dashboard
     - oncall rotation 등록
     - 표준 IAM Role
  3. 개발자는 코드만 작성

→ 가장 쉬운 길이 표준 / 안전 / 최적.
```

→ 일탈 ("내 마음대로") 도 가능하지만 결국 cost ↑.

---

## 5. 하위 영역

- [[idp-concepts]] — IDP 디자인 원칙
- [[backstage]] — Spotify 의 OSS IDP
- [[crossplane]] — k8s 위의 cloud provisioning
- [[scaffolding]] — service template / cookiecutter
- [[self-service]] — UI / CLI / GitOps
- [[developer-experience]] — DX metric
- [[pitfalls]]

---

## 6. 도구 (IDP)

| | type | 강점 |
| --- | --- | --- |
| **Backstage** | OSS (Spotify) | 표준, plugin 많음 |
| **Port** | SaaS | UI 친화 |
| **Cortex** | SaaS | service catalog |
| **OpsLevel** | SaaS | scorecard |
| **Crossplane** | OSS k8s | infra provisioning |
| **Humanitec** | SaaS | platform orchestrator |
| **Cycloid** | SaaS | DevOps tool 통합 |
| **Roadie** | Backstage SaaS | managed |
| **Atlassian Compass** | SaaS | service catalog |

→ **첫 도입 = Backstage** (커뮤니티 강함).

---

## 7. 학습 순서

1. Day 1 — [[idp-concepts]] (IDP 가 뭔지, golden path)
2. Day 2 — [[scaffolding]] (template / cookiecutter)
3. Day 3 — [[backstage]] (설치 + plugin)
4. Day 4 — [[crossplane]] (infra-as-code 추상화)
5. Day 5 — [[developer-experience]] (DX metric)

---

## 8. 책 / 자료

- **"Team Topologies"** (Skelton/Pais) — platform team 디자인
- **"Platform Engineering on Kubernetes"** (Salatino)
- CNCF Platforms Working Group
- platformengineering.org

---

## 9. 관련

- [[../devops|↑ devops]]
- [[../cicd/cicd|↗ CI/CD]]
- [[../iac/iac|↗ IaC]]
- [[../sre/sre|↗ SRE]]
