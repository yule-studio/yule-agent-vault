---
title: "Matrix builds — OS / 언어 버전 매트릭스"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:42:00+09:00
tags: [devops, cicd, matrix]
---

# Matrix builds — OS / 언어 버전 매트릭스

**[[cicd|↑ cicd]]**

---

## 1. GitHub Actions

```yaml
strategy:
  fail-fast: false
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
    java: ['17', '21']
    include:
      - {os: ubuntu-latest, java: '23'}     # 추가
    exclude:
      - {os: windows-latest, java: '17'}    # 제외

runs-on: ${{ matrix.os }}
steps:
  - uses: actions/setup-java@v4
    with: {java-version: '${{ matrix.java }}', distribution: temurin}
  - run: ./gradlew test
```

→ 3 OS × 2 java = 6 jobs (병렬).

---

## 2. GitLab CI

```yaml
test:
  parallel:
    matrix:
      - JAVA_VERSION: ['17', '21']
        OS: [ubuntu, macos]
  image: openjdk:$JAVA_VERSION-$OS
  script: ./gradlew test
```

---

## 3. CircleCI

```yaml
workflows:
  test:
    jobs:
      - test:
          matrix:
            parameters:
              java-version: ["17", "21"]
              os: ["ubuntu", "macos"]
```

---

## 4. 함정

1. **matrix 폭증** — 3 × 3 × 2 × 2 = 36 jobs.
2. **fail-fast: true** — 1 fail 시 모두 cancel.
3. **OS 별 path 차이** — Linux/Mac/Windows path separator.
4. **artifact 충돌** — matrix job 별 unique name.
5. **비용 spike** (paid 시).

---

## 관련

- [[cicd|↑ cicd]]
- [[github-actions]]
