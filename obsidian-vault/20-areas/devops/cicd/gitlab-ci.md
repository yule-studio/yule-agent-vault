---
title: "GitLab CI"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:22:00+09:00
tags: [devops, cicd, gitlab-ci]
---

# GitLab CI

**[[cicd|↑ cicd]]**

---

## 1. `.gitlab-ci.yml`

```yaml
stages: [build, test, scan, deploy]

variables:
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

cache:
  paths:
    - .gradle/caches/

build:
  stage: build
  image: eclipse-temurin:21-jdk
  script:
    - ./gradlew bootJar
  artifacts:
    paths: [build/libs/*.jar]
    expire_in: 1 day

test:
  stage: test
  image: eclipse-temurin:21-jdk
  script:
    - ./gradlew test
  artifacts:
    reports:
      junit: build/test-results/test/*.xml

build-image:
  stage: build
  image: docker:24
  services: [docker:24-dind]
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE .
    - docker push $IMAGE

scan:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - trivy image --severity CRITICAL,HIGH --exit-code 1 $IMAGE

deploy-prod:
  stage: deploy
  image: bitnami/kubectl
  script:
    - kubectl set image deploy/web app=$IMAGE
    - kubectl rollout status deploy/web
  environment:
    name: production
    url: https://api.example.com
  when: manual
  only: [main]
```

---

## 2. environment + 보호

- `environment: production` 정의 → GitLab UI 에 "Production" 환경 표시.
- 보호 — `Settings → CI/CD → Protected environments` 에서 user/role 별 deploy 허용.

---

## 3. self-hosted runner

```bash
# Linux
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt install gitlab-runner

# register
sudo gitlab-runner register \
    --url https://gitlab.com/ \
    --registration-token <TOKEN>
```

---

## 4. include (재사용)

```yaml
include:
  - project: 'group/templates'
    ref: main
    file: '/ci/test.yml'
  - remote: 'https://example.com/ci-common.yml'
```

---

## 5. GitLab vs GitHub Actions

| 항목 | GitHub Actions | GitLab CI |
| --- | --- | --- |
| YAML 구조 | workflow → jobs → steps | stages → jobs → script |
| Reusable | reusable workflow | include / extends |
| Self-hosted runner | O | O |
| 통합 IDE | GitHub Codespaces | GitLab Web IDE |
| Container registry | GHCR | GitLab Container Registry |

---

## 6. 함정

1. **services: docker:dind + privileged** → 보안.
2. **artifact 영구** → 디스크 폭주 → expire_in.
3. **runner 1대 share** → bottleneck.
4. **protected branch 의 secret** → 우회 시 leak.

---

## 7. 관련

- [[cicd|↑ cicd]]
- [[tools-comparison]]
- [[github-actions]]
