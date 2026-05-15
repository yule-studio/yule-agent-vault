---
title: "Backstage — Spotify 의 OSS IDP"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:26:00+09:00
tags: [devops, platform-engineering, backstage]
---

# Backstage — Spotify 의 OSS IDP

**[[platform-engineering|↑ platform-engineering]]**

---

## 1. 무엇

Spotify 가 만든 OSS developer portal:
- service catalog (모든 service 의 metadata)
- TechDocs (docs)
- software templates (scaffolding)
- plugin (1000+)
- Software Catalog (entity model)

→ CNCF Incubating project. enterprise 표준.

---

## 2. 설치

```bash
# 새 app 생성
npx @backstage/create-app@latest --path my-backstage

cd my-backstage
yarn dev          # http://localhost:3000
```

→ 처음 startup 만 30분, 그 후 plugin 추가가 큰 작업.

---

## 3. core 구조

```
backstage/
├── packages/
│   ├── app/             # frontend (React)
│   └── backend/         # backend (Node.js)
├── plugins/
└── app-config.yaml      # 전체 config
```

---

## 4. Entity (Software Catalog)

```yaml
# catalog-info.yaml — 각 service 의 repo 에 위치
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: user-service
  description: User service for signup/login
  annotations:
    github.com/project-slug: myorg/user-service
    jenkins.io/job-full-name: user-service-build
    pagerduty.com/integration-key: xxx
  tags: [java, spring, prod]
  links:
    - url: https://grafana.example.com/dashboard/user
      title: Dashboard
spec:
  type: service             # service / website / library / documentation
  lifecycle: production     # experimental / production / deprecated
  owner: team-platform
  system: payment
  dependsOn:
    - resource:postgres-prod
    - component:notification-service
```

→ Backstage 가 git scan 으로 자동 발견 (또는 GitHub API).

---

## 5. Entity 종류

| Kind | 무엇 |
| --- | --- |
| **Component** | service / app |
| **API** | OpenAPI / GraphQL schema |
| **Resource** | DB / queue / bucket |
| **System** | 여러 component 의 그룹 |
| **Domain** | system 의 비즈니스 영역 |
| **Group** | 팀 |
| **User** | 직원 |
| **Location** | catalog-info 파일 위치 |
| **Template** | scaffolding template |

---

## 6. Software Templates (scaffolding ★)

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata: {name: spring-boot-service, title: Spring Boot Service}
spec:
  owner: team-platform
  type: service
  parameters:
    - title: 기본 정보
      properties:
        name: {type: string, title: 서비스 이름}
        description: {type: string}
        owner: {type: string, title: owner team}

  steps:
    - id: fetch-template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          owner: ${{ parameters.owner }}

    - id: publish
      action: publish:github
      input:
        repoUrl: github.com?owner=myorg&repo=${{ parameters.name }}

    - id: register
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml

  output:
    links:
      - {title: Repository, url: ${{ steps.publish.output.remoteUrl }}}
      - {title: Open in catalog, url: ${{ steps.register.output.entityRef }}}
```

→ Backstage UI 에서 "Create Service" 클릭 → 30초 안에 새 repo + CI + catalog.

---

## 7. plugin 예

| | 무엇 |
| --- | --- |
| **TechDocs** | markdown 문서를 service 옆에 |
| **Kubernetes** | service 의 k8s pod / status |
| **GitHub Actions / Jenkins** | CI 상태 |
| **PagerDuty / Opsgenie** | on-call |
| **SonarQube** | code quality |
| **Grafana / Prometheus** | metric |
| **ArgoCD** | GitOps state |
| **AWS / Cost Insights** | cloud cost |
| **Cortex / OpsLevel scorecard** | 자체 |

---

## 8. TechDocs

```yaml
# mkdocs.yml (in repo)
site_name: User Service
nav:
  - Home: index.md
  - API: api.md
  - Runbook: runbook.md

# 자동 build → Backstage 의 service page 에서 보임
```

```yaml
# catalog-info.yaml
metadata:
  annotations:
    backstage.io/techdocs-ref: dir:.
```

→ docs 가 코드 옆 (git). Backstage 가 build + serve.

---

## 9. permission

```yaml
# 기본 — 모두 read
# 수정 / 삭제 — 권한
permissions:
  - allow:
      action: catalog.entity.create
      conditions: [{rule: HAS_LABEL, params: {key: owner}}]
```

```yaml
# Auth (Google / GitHub / Okta)
auth:
  environment: production
  providers:
    google:
      production:
        clientId: ${AUTH_GOOGLE_CLIENT_ID}
        clientSecret: ${AUTH_GOOGLE_CLIENT_SECRET}
```

---

## 10. scorecard

```yaml
# 어떤 service 가 표준 충족?
scorecard:
  - name: production-readiness
    rules:
      - description: "README 있음"
        condition: hasFile(/README.md)
      - description: "monitoring dashboard"
        condition: hasAnnotation(grafana.com/dashboard-url)
      - description: "on-call 등록"
        condition: hasAnnotation(pagerduty.com/integration-key)
      - description: "SLO 정의"
        condition: hasFile(/slo.yaml)
      - description: "최근 6개월 deploy"
        condition: lastDeployWithin(180d)
```

→ "production-ready 8/10" 식 점수.

---

## 11. 운영 (host)

```
방법:
  1. self-host on k8s (helm chart 있음) — 무료 OSS
  2. Roadie (SaaS Backstage)
  3. Spotify Portal (Spotify 의 hosted)

규모:
  - frontend stateless (replica N)
  - backend stateless + DB (Postgres for catalog/cache)
```

---

## 12. 함정

1. **catalog 안 채우면 빈 UI** — 자동 discovery 필수.
2. **plugin 너무 많음** — 무거움 / 충돌.
3. **app-config.yaml 비대** — 환경별 분리.
4. **upgrade 어려움** — major version 마다 breaking.
5. **자체 plugin 작성** — TypeScript + React 학습.
6. **권한 정책 없음** — 누구나 catalog 삭제.

---

## 13. 관련

- [[platform-engineering|↑ platform-engineering]]
- [[scaffolding]]
- [[idp-concepts]]
