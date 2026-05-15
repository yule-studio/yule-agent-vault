---
title: "CircleCI — SaaS"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:26:00+09:00
tags: [devops, cicd, circleci]
---

# CircleCI — SaaS

**[[cicd|↑ cicd]]**

---

## 1. `.circleci/config.yml`

```yaml
version: 2.1

orbs:
  docker: circleci/docker@2.5

executors:
  java:
    docker:
      - image: cimg/openjdk:21.0

jobs:
  test:
    executor: java
    steps:
      - checkout
      - restore_cache:
          keys: [gradle-{{ checksum "build.gradle" }}]
      - run: ./gradlew test
      - save_cache:
          key: gradle-{{ checksum "build.gradle" }}
          paths: [~/.gradle/caches]
      - store_test_results:
          path: build/test-results

  build-push:
    executor: docker/docker
    steps:
      - checkout
      - setup_remote_docker
      - docker/check
      - docker/build:
          image: myorg/myapp
          tag: ${CIRCLE_SHA1}
      - docker/push:
          image: myorg/myapp
          tag: ${CIRCLE_SHA1}

workflows:
  ci:
    jobs:
      - test
      - build-push:
          requires: [test]
          filters:
            branches: { only: main }
```

---

## 2. orb (재사용)

- CircleCI 의 plugin — Docker / AWS / Slack / Jira 등.
- 사용: `orbs: docker: circleci/docker@2.5`.

---

## 3. matrix

```yaml
test:
  parameters:
    java-version: { type: string }
  executor:
    name: java
    java-version: << parameters.java-version >>
  steps: [...]

workflows:
  ci:
    jobs:
      - test:
          matrix:
            parameters:
              java-version: ["17", "21"]
```

---

## 4. cache

- restore_cache + save_cache (cache key).
- workspace (job 간 file 전달).

---

## 5. CircleCI vs GitHub Actions

| | CircleCI | GitHub Actions |
| --- | --- | --- |
| 설정 file | `.circleci/config.yml` | `.github/workflows/*.yml` |
| reuse | orb | reusable workflow / action |
| 가격 | $30/month 기본 | free (한도) / pay-as-go |
| 빠른 SaaS | ★ | (느릴 수 있음) |
| GitHub 통합 | OK (외부) | native |

→ GitHub 사용 시 GitHub Actions 우선.

---

## 6. 관련

- [[cicd|↑ cicd]]
- [[tools-comparison]]
