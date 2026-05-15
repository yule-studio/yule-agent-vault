---
title: "GitHub Actions ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:19:00+09:00
tags: [devops, cicd, github-actions]
---

# GitHub Actions ★

**[[cicd|↑ cicd]]**

---

## 1. 구조

```
.github/
└── workflows/
    ├── ci.yml            # PR / push 시
    ├── release.yml       # tag 시
    └── nightly.yml       # 스케줄
```

---

## 2. 기본 workflow (Spring Boot)

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  packages: write     # GHCR push

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: gradle
      - run: ./gradlew test
      - uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Tests
          path: '**/build/test-results/test/*.xml'
          reporter: java-junit

  build-push:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=sha-,format=short
            type=raw,value=latest,enable={{is_default_branch}}
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  scan:
    needs: build-push
    runs-on: ubuntu-latest
    steps:
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:sha-${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: 1
```

---

## 3. matrix builds (다중 OS / 버전)

```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        java: ['17', '21']
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-java@v4
        with: {java-version: '${{ matrix.java }}'}
      - run: ./gradlew test
```

---

## 4. secret + environment

```yaml
jobs:
  deploy-prod:
    environment: production    # GitHub UI 에서 approval / restriction
    runs-on: ubuntu-latest
    steps:
      - run: echo "${{ secrets.PROD_API_KEY }}"
```

→ `Settings → Environments` 에서 secret + approver 설정.

---

## 5. reusable workflow

```yaml
# .github/workflows/reusable-test.yml
on:
  workflow_call:
    inputs:
      java-version:
        type: string
        default: '21'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: {java-version: '${{ inputs.java-version }}'}
      - run: ./gradlew test
```

```yaml
# 호출
jobs:
  test:
    uses: ./.github/workflows/reusable-test.yml
    with: {java-version: '21'}
```

---

## 6. self-hosted runner

```bash
# repo Settings → Actions → Runners → New self-hosted
# 호스트에서 install + start
./run.sh
```

```yaml
runs-on: [self-hosted, linux, gpu]
```

→ GPU / 네트워크 격리 / 큰 cache.

---

## 7. cost / 한도

| Plan | minutes / month | concurrent |
| --- | --- | --- |
| Free (public) | 무한 | 20 |
| Free (private) | 2000 | 20 |
| Pro | 3000 | 40 |
| Team | 3000 | 60 |
| Enterprise | 50000 | 180 |

→ 무료 minutes 초과 시 분당 $0.008 (Linux).

---

## 8. 함정

1. **secret echo** → log 누설 → `${{ secrets.X }}` 는 자동 mask but `printenv` 등 skip 가능.
2. **fork PR + secret** → secret 자동 X (보안).
3. **cache miss** — key 변경 / dependency 변경 시.
4. **self-hosted runner 보안** — public repo 면 PR 이 malicious code 실행 가능.
5. **matrix 폭증** — 3 OS × 3 java × 2 region = 18 jobs.

---

## 9. 관련

- [[cicd|↑ cicd]]
- [[tools-comparison]]
- [[pipeline-patterns]]
