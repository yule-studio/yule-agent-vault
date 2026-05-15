---
title: "Scaffolding — service template"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:30:00+09:00
tags: [devops, platform-engineering, scaffolding]
---

# Scaffolding — service template

**[[platform-engineering|↑ platform-engineering]]**

---

## 1. 왜

새 service 시작:
```
- git repo
- gradle / package.json
- Dockerfile
- CI/CD pipeline
- k8s manifest
- monitoring config
- README / runbook
- 표준 dependency
- code style / lint
- pre-commit hooks
- ...
```

→ 새 service 시작 = 1-2일 부담. Template 으로 30분 ↓.

→ 일관성 + best practice 강제.

---

## 2. 도구

| | 무엇 |
| --- | --- |
| **Backstage Software Templates** | UI / git / k8s 통합 |
| **Cookiecutter** | Python, 단순 |
| **Yeoman** | Node, 풍부 |
| **JHipster** | Spring 전용, 강력 |
| **Spring Initializr** | Spring start 표준 |
| **create-react-app / Vite** | React |
| **Helm chart template** | k8s 표준 |
| **Hygen** | code generator |
| **Plop.js** | code generator |

→ 가장 표준 = Backstage Software Template.

---

## 3. Cookiecutter (간단)

```
my-template/
├── cookiecutter.json
└── {{cookiecutter.project_slug}}/
    ├── README.md
    ├── pom.xml          (또는 build.gradle.kts)
    ├── Dockerfile
    └── src/...

# cookiecutter.json
{
    "project_name": "My Service",
    "project_slug": "{{ cookiecutter.project_name|lower|replace(' ', '-') }}",
    "owner_team": "platform",
    "java_version": "21",
    "uses_postgres": "yes"
}
```

```bash
cookiecutter ./my-template
# → 입력 prompt → repo 생성
```

---

## 4. Backstage Template 예 (Spring Boot)

```
spring-boot-template/
├── template.yaml          # Template 정의
└── skeleton/              # 실제 코드
    ├── ${{ values.name }}/
    │   ├── build.gradle.kts
    │   ├── Dockerfile
    │   ├── .github/workflows/ci.yml
    │   ├── k8s/
    │   │   ├── deployment.yaml
    │   │   └── service.yaml
    │   ├── src/main/kotlin/com/example/${{ values.name }}/
    │   │   └── Application.kt
    │   ├── catalog-info.yaml
    │   └── mkdocs.yml
```

`template.yaml`:
```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata: {name: spring-boot, title: Spring Boot}
spec:
  parameters:
    - title: Service 정보
      required: [name, owner, system]
      properties:
        name:
          type: string
          pattern: '^[a-z][a-z0-9-]*$'
        owner:
          type: string
          ui:field: OwnerPicker
          ui:options: {allowedKinds: [Group]}
        system:
          type: string
          ui:field: EntityPicker
          ui:options: {allowedKinds: [System]}

    - title: 추가 설정
      properties:
        database: {type: string, enum: [none, postgres, mysql]}
        cache: {type: boolean}

  steps:
    - id: fetch
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          owner: ${{ parameters.owner }}
          database: ${{ parameters.database }}

    - id: publish
      action: publish:github
      input:
        repoUrl: github.com?repo=${{ parameters.name }}&owner=myorg
        defaultBranch: main
        description: ${{ parameters.name }} - generated from spring-boot template

    - id: register
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml
```

→ Backstage UI 에서 "Create" → 30초 안에 새 repo + CI + catalog.

---

## 5. template 의 필수 요소

```
필수:
  - README.md (사용법)
  - CI/CD (.github/workflows or .gitlab-ci.yml)
  - Dockerfile
  - k8s manifest (또는 Helm chart)
  - .gitignore
  - 라이센스
  - catalog-info.yaml (Backstage)

권장:
  - mkdocs.yml + docs/
  - pre-commit hook
  - linter / formatter
  - test setup
  - .env.example
  - runbook.md
  - SECURITY.md
```

---

## 6. variable 처리

```
template engine:
  - Nunjucks (Backstage)
  - Jinja2 (Cookiecutter)
  - Mustache
  - Go template

placeholder:
  - ${{ values.name }}        (Backstage)
  - {{ cookiecutter.name }}   (Cookiecutter)
  - {{.Values.name}}          (Helm)
```

---

## 7. validate (CI)

```yaml
# template repo 의 자체 CI
- name: validate template
  run: |
    backstage-cli template validate template.yaml
    # 또는 cookiecutter render + tests
```

→ template 자체도 코드 — 검증 / test.

---

## 8. evolution

```
template upgrade:
  - 새 version (v2) 출시
  - 기존 service 는 v1 사용 — break X
  - 사용자가 "migrate" 액션 (자동 PR)
  - 점진 마이그
```

→ template 도 product (version + roadmap).

---

## 9. 함정

1. **template 만 만들고 안 갱신** — stale → 옛 best practice 강제.
2. **너무 많은 variant** — fragmentation. 1-3 권장.
3. **너무 적음** — 다양 case 못 다룸.
4. **upgrade path 없음** — 옛 service 들이 stale.
5. **변경 후 강제** — 사용자 반발. opt-in 부터.
6. **template 자체 test 없음** — render 시 깨짐.

---

## 10. 관련

- [[platform-engineering|↑ platform-engineering]]
- [[backstage]]
- [[idp-concepts]]
- [[../cicd/cicd|↗ CI/CD]]
