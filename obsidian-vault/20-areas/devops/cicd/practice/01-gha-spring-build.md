---
title: "실습 01 — GitHub Actions Spring build + push GHCR"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:50:00+09:00
tags: [devops, cicd, practice, github-actions]
---

# 실습 01 — GitHub Actions Spring build + push GHCR

**[[practice|↑ practice]]**

> 30분 — push 만 하면 image 가 GHCR 에 자동 push.

---

## 1. 준비

- GitHub repo (public 또는 private).
- Dockerfile ([[../../docker/practice/02-dockerfile-spring-boot]] 의 것).
- `.github/workflows/` 디렉토리.

---

## 2. workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read
  packages: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: gradle

      - run: ./gradlew test

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        if: github.event_name != 'pull_request'
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
```

---

## 3. 실행 확인

```
push main
  ↓
Actions 탭 → CI workflow 실행
  ↓
Packages 탭 → 새 image: ghcr.io/USER/REPO:latest + sha-xxxx
```

```bash
# 로컬에서 pull
docker pull ghcr.io/USER/REPO:latest
```

---

## 4. 다음

[[02-gha-k8s-deploy]]
