---
title: "Buildkite — hybrid (SaaS UI + self-hosted agent)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:28:00+09:00
tags: [devops, cicd, buildkite]
---

# Buildkite — hybrid (SaaS UI + self-hosted agent)

**[[cicd|↑ cicd]]**

---

## 1. 무엇

- **UI / orchestration = SaaS** (buildkite.com).
- **agent = self-hosted** (호스트 / 컨테이너 / k8s).
- 보안 (코드 / artifact 가 자체 인프라).
- 큰 monorepo 의 분산 build.

---

## 2. `.buildkite/pipeline.yml`

```yaml
steps:
  - label: "Test"
    command: "./gradlew test"
    agents:
      queue: "default"
    plugins:
      - docker#v5.10.0:
          image: "eclipse-temurin:21-jdk"

  - wait

  - label: "Build + push"
    command: |
      docker build -t myorg/myapp:$BUILDKITE_COMMIT .
      docker push myorg/myapp:$BUILDKITE_COMMIT
    agents:
      queue: "docker"

  - wait

  - block: "Deploy production?"
    branches: "main"

  - label: "Deploy"
    command: "kubectl set image deploy/web app=myorg/myapp:$BUILDKITE_COMMIT"
    branches: "main"
```

---

## 3. agent 종류

- **systemd** (linux 서버)
- **Docker** (컨테이너 안)
- **k8s** (controller + pod 동적)
- **AWS EC2 plugin** — autoscaling

---

## 4. dynamic pipeline

```yaml
steps:
  - command: "buildkite-agent pipeline upload < dynamic-pipeline.yml"
```

→ 코드 의 결과를 기반으로 다음 step 동적 생성.

---

## 5. parallelism

```yaml
steps:
  - label: "Test"
    command: "./gradlew test --tests=ShardTest$BUILDKITE_PARALLEL_JOB"
    parallelism: 10
```

→ 10 개 agent 분산.

---

## 6. 함정

1. **agent 의 secret** — host 에 저장 → 적절한 권한.
2. **dynamic pipeline 의 무한 loop** → 자체 제한 필요.
3. **monorepo path filter** — buildkite-monorepo-diff plugin.

---

## 7. 관련

- [[cicd|↑ cicd]]
- [[tools-comparison]]
