---
title: "Pipeline cache — dependency / build"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:40:00+09:00
tags: [devops, cicd, cache]
---

# Pipeline cache — dependency / build

**[[cicd|↑ cicd]]**

---

## 1. 종류

| 종류 | 효과 | 도구 |
| --- | --- | --- |
| **dependency cache** | gradle / npm / pip download skip | actions/cache (GH) |
| **build cache** | gradle / sccache / bazel | gradle remote cache |
| **Docker layer cache** | image rebuild ↓ | BuildKit `--cache-from` |
| **artifact** | job 간 file 전달 | actions/upload-artifact |

---

## 2. GitHub Actions

```yaml
# gradle
- uses: actions/setup-java@v4
  with:
    distribution: temurin
    java-version: '21'
    cache: gradle

# npm
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'

# Custom
- uses: actions/cache@v4
  with:
    path: ~/.m2
    key: m2-${{ hashFiles('**/pom.xml') }}
    restore-keys: m2-
```

---

## 3. Docker BuildKit cache

```yaml
- uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

→ GitHub Actions cache 에 Docker layer 저장.

---

## 4. Gradle remote cache (큰 team)

```kotlin
// settings.gradle.kts
buildCache {
    remote<HttpBuildCache> {
        url = uri("https://gradle-cache.internal/")
        credentials {
            username = System.getenv("GRADLE_CACHE_USER")
            password = System.getenv("GRADLE_CACHE_PASS")
        }
        push = System.getenv("CI") == "true"
    }
}
```

---

## 5. 함정

1. **cache key 변경 X** → 옛 cache 영원히.
2. **cache size 큼** (1GB+) → 저장 / 다운로드 비용.
3. **secret 이 cache 에** → 누설 위험.
4. **stale cache** — dependency update 후 옛 lock.

---

## 관련

- [[cicd|↑ cicd]]
- [[pipeline-patterns]]
