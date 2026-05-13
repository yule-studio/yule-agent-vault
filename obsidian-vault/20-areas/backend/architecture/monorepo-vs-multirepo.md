---
title: "Monorepo vs Multi-repo — 결정 / 도구 / 마이그레이션"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:15:00+09:00
tags:
  - backend
  - architecture
  - monorepo
  - multirepo
---

# Monorepo vs Multi-repo — 결정 / 도구 / 마이그레이션

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 결정 표 / Nx·Bazel·Turborepo·Gradle / 마이그레이션 |

**[[architecture|↑ architecture hub]]**

> "MSA = multi-repo" 는 **틀린 등식**. Google 은 거대 monorepo + MSA. Netflix 는 multi-repo + MSA.
> **배포 단위 결정** 과 **repo 결정** 은 직교 축. 이 문서는 repo 결정만.

---

## 1. 한 줄 정의

| | Monorepo | Multi-repo |
| --- | --- | --- |
| **정의** | 모든 코드를 하나의 git 저장소 | 컴포넌트마다 별도 저장소 |
| **빌드** | 도구로 affected 만 빌드 | 각자 알아서 |
| **버전** | trunk-based / 단일 버전 | semver / 의존성 lockfile |
| **권한** | path-based 또는 단일 (다 보임) | repo-level 권한 분리 |
| **대표 회사** | Google / Meta / Uber / Microsoft (일부) | Netflix / Amazon / 대부분 한국 회사 |

---

## 2. 결정 표 — 어느 쪽?

| 기준 | Monorepo 가 유리 | Multi-repo 가 유리 |
| --- | --- | --- |
| **공유 코드 빈도** | 자주 공유 (UI / 도메인 / proto) | 거의 안 공유 |
| **팀 협업** | 같은 코드 자주 봄 | 팀별 완전 독립 |
| **빌드 도구** | 도입 가능 (Bazel/Nx/...) | 단순 (각자 CI) |
| **버전 관리** | atomic commit 필요 (e.g. shared schema 변경) | 컴포넌트 별 SemVer |
| **권한 / 컴플라이언스** | 회사 전체 공유 OK | 외주 / 고객사 분리 필요 |
| **저장소 크기** | 작거나 도구로 감당 (~수십 GB) | 거대해도 분산 |
| **CI 비용** | 도구 없으면 폭발 | 자연스럽게 분리 |
| **신규 서비스 생성 비용** | 폴더 하나 (10초) | repo 하나 + CI + secret + IAM (수시간) |
| **외부 공개 일부만** | submodule / 별 repo | 자연스러움 |

**짧은 결정 룰**:
- 팀 ≤ 30명 + 공유 코드 있음 → **Monorepo + 도구 (Nx/Turborepo)**
- 팀 ≥ 50명 + 비대칭 컴포넌트 + 외부 노출 SDK → **Multi-repo or Hybrid**
- "MSA 인데 어디로?" → 둘 다 OK. 팀 자율성 우선이면 multi, 일관성 우선이면 mono.

---

## 3. 장단점 깊이

### 3.1 Monorepo 의 진짜 이득

- **Atomic Refactor**: shared interface 변경 + 모든 콜러 동시 수정을 **1 PR**. multi-repo 는 PR 사슬.
- **하나의 진실의 원천**: 누가 무엇을 쓰는지 `git grep` 한 번.
- **일관 도구**: 빌드 / 린트 / fmt / CI 한 곳.
- **신규 서비스 진입 0초**: 폴더 + 빌드 타겟 추가.

### 3.2 Monorepo 의 진짜 비용

- **빌드 도구 의존**: Nx / Bazel / Turborepo 없이는 CI 시간 폭발.
- **거대 저장소**: 클론 수 GB+. **git LFS / sparse-checkout / partial clone** 필수.
- **권한 분리 어려움**: 누구나 모든 코드 봄. `CODEOWNERS` 로 일부 제한.
- **변경 영향 범위 추적**: "이 함수 누가 쓰는지" 한 번에 보임 = 변경 책임 ↑.

### 3.3 Multi-repo 의 진짜 이득

- **완전 격리**: 한 repo 의 빌드 깨짐이 다른 repo 영향 X.
- **권한 자연스러움**: GitHub repo level 권한.
- **다른 언어 / 다른 CI 자연스러움**: 각자 best practice.
- **외부 공유 쉬움**: SDK / 클라이언트 라이브러리.

### 3.4 Multi-repo 의 진짜 비용

- **공유 코드 = 라이브러리 + SemVer**: 변경마다 publish → 사용처 업데이트 PR 폭발.
- **저장소 폭발**: 100+ repo 가 되면 "이 함수 어디 있지?" 가 5분짜리 수색.
- **표준 일관성 어려움**: lint / dependency / CI 가 repo 마다 drift.
- **PR 사슬 (chained PRs)**: 의존성 sshared lib v2 → service A 의존성 → service B 의존성 = 3 PR + 순서 / 시간 차.

---

## 4. 도구 — Monorepo

### 4.1 도구 비교 (2026)

| 도구 | 언어 | 강점 | 약점 |
| --- | --- | --- | --- |
| **Nx** | TS/JS 중심, polyglot 가능 | DX 좋음, plugin 풍부, affected 잘 | TS 외 깊이 떨어짐 |
| **Turborepo** | TS/JS | 가볍고 빠름, Vercel 통합 | JS 외 X, Nx 대비 단순 |
| **Bazel** | polyglot (Google) | 거대 규모, 빠름, hermetic | 학습 곡선 가파름 |
| **Pants** | polyglot, Python 강함 | Bazel 보다 친절 | 생태계 작음 |
| **Gradle (composite build)** | JVM | Spring 진영 자연스러움 | JVM 외 X |
| **Lerna / Rush** | TS/JS | 옛 표준 | Nx / Turborepo 에 밀림 |
| **Buck2** | polyglot (Meta) | 빠름, Bazel 유사 | 생태계 작음 |

### 4.2 Nx (가장 표준적인 선택)

```bash
# 신규 monorepo
npx create-nx-workspace@latest myorg --preset=ts

# 기존 monorepo 에 패키지 추가
nx g @nx/js:lib shared-domain
nx g @nx/node:app order-service
nx g @nx/node:app payment-service

# affected 만 빌드 / 테스트
nx affected --target=test --base=main --head=HEAD
nx affected --target=build
nx affected --target=lint

# 그래프 시각화
nx graph
```

`nx.json` 예시:

```json
{
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx-cloud",
      "options": {
        "cacheableOperations": ["build", "test", "lint"],
        "accessToken": "..."
      }
    }
  },
  "targetDefaults": {
    "build": { "dependsOn": ["^build"], "inputs": ["production"] },
    "test":  { "inputs": ["default", "^production"] }
  },
  "namedInputs": {
    "default":    ["{projectRoot}/**/*", "sharedGlobals"],
    "production": ["default", "!{projectRoot}/**/*.spec.ts"],
    "sharedGlobals": []
  }
}
```

### 4.3 Bazel (구글 식 — 거대 규모 polyglot)

```python
# WORKSPACE
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")
# ... 의존성

# services/order/BUILD.bazel
java_binary(
    name = "order-service",
    srcs = glob(["src/main/java/**/*.java"]),
    deps = [
        "//libs/domain-common:common",
        "@maven//:org_springframework_boot_spring_boot_starter_web",
    ],
    main_class = "com.example.OrderApp",
)

java_test(
    name = "OrderServiceTest",
    srcs = ["src/test/java/com/example/OrderServiceTest.java"],
    deps = [":order-service", "@maven//:junit_jupiter_api"],
)
```

```bash
# affected 빌드
bazel build //services/order/...
bazel test //...
bazel query 'rdeps(//..., //libs/domain-common:common)'    # 누가 의존하는지
```

> **Bazel 함정**: 처음 학습 곡선 가파름. 30명 이하 팀은 Nx / Turborepo 권장.

### 4.4 Turborepo (단순한 JS/TS monorepo)

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "test":  { "outputs": [] },
    "dev":   { "cache": false, "persistent": true }
  }
}
```

```bash
turbo run build --filter=...^order-service     # order-service 와 그 의존성만
turbo run test --filter='[origin/main]'        # main 대비 변경된 것만
```

### 4.5 Gradle Composite Build (JVM monorepo)

```kotlin
// settings.gradle.kts
rootProject.name = "myorg"

include(":libs:domain-common")
include(":libs:shared-events")
include(":services:order")
include(":services:payment")
include(":services:user")

// 또는 includeBuild 로 더 큰 분리
includeBuild("../other-monorepo")
```

```bash
./gradlew :services:order:build              # order 만
./gradlew test                                # 모두
./gradlew :services:order:dependencies
```

**JVM 진영의 Spring monorepo 첫 선택지**.

### 4.6 폴더 구조 — Nx 식 polyglot 예시

```
myorg/
├── apps/                              # 배포 단위
│   ├── order-service/                 # @myorg/order-service
│   ├── payment-service/
│   ├── user-service/
│   ├── web-customer/                  # Next.js
│   └── web-admin/
├── libs/                              # 공유 / 재사용
│   ├── domain/
│   │   ├── order/                     # order 도메인 (POJO + use cases)
│   │   └── payment/
│   ├── shared/
│   │   ├── events/                    # Kafka schema (proto / avro / TS types)
│   │   ├── api-contract/              # OpenAPI / gRPC proto
│   │   └── ui-kit/                    # React component
│   └── infrastructure/
│       └── observability/
├── tools/                             # CI / 스크립트 / generators
├── nx.json
├── package.json
└── tsconfig.base.json
```

### 4.7 폴더 구조 — Gradle 식 JVM 예시

```
myorg/
├── settings.gradle.kts
├── build.gradle.kts                   # 루트 (공통 의존성)
├── buildSrc/                          # 빌드 로직 공유
├── libs/
│   ├── domain-common/
│   ├── shared-events/
│   └── api-contract/                  # OpenAPI 생성
└── services/
    ├── order/
    ├── payment/
    └── user/
```

---

## 5. 도구 — Multi-repo

### 5.1 도구 / 패턴

| 영역 | 도구 |
| --- | --- |
| 코드 검색 | Sourcegraph / GitHub search |
| 종속성 업데이트 | Renovate / Dependabot |
| 공유 라이브러리 | npm registry / Maven Central / Nexus / Artifactory |
| 표준화 | OpenSSF Scorecard / cookiecutter / scaffold |
| 다중 PR 조정 | depend-on-me / git-stack |
| repo 일괄 변경 | gh CLI + 스크립트 / Atlantis |

### 5.2 표준 repo template

```
myorg/order-service/
├── .github/
│   ├── workflows/ci.yml
│   ├── CODEOWNERS
│   ├── PULL_REQUEST_TEMPLATE.md
│   └── ISSUE_TEMPLATE/
├── docs/
├── src/main/...
├── src/test/...
├── Dockerfile
├── README.md
└── build.gradle.kts
```

새 repo 생성 = template 복사. cookiecutter / GitHub template repo / Backstage software template 권장.

### 5.3 공유 라이브러리 SemVer

```
common-events  1.0.0 → 1.1.0 (additive)  → 2.0.0 (breaking)
                 ↓               ↓               ↓
order-service  1.0.0 → 1.1.0 (consume)  → 2.0.0 (rewrite)
payment-...    1.0.0 → 1.0.0 (still)    → 2.0.0 (rewrite)
```

**Breaking change 시 모든 consumer rewrite 비용**. 그래서 monorepo 의 "한 번에 수정" 이 강력.

방어:
- additive only 정책 (deprecated 후 2 분기 지나서 제거)
- API 검증 도구 — `japicmp` (Java), `api-extractor` (TS)

---

## 6. Hybrid — 자주 보는 절충안

```
myorg-core (monorepo)              myorg-sdk-js (multi)
├── services/                       
│   ├── order, payment, user...      myorg-public-api (multi)
├── libs/internal                    
└── ...                              myorg-customer-portal (multi)
```

- **core monorepo** = 내부 서비스 + 공유 도메인
- **별 repo** = 외부 공개 SDK / 고객용 UI / 외주 작업

→ 내부 atomic 의 이점 + 외부 격리.

---

## 7. 마이그레이션

### 7.1 Multi-repo → Monorepo

**도구**: `git filter-repo` (powerful) 또는 `git subtree`.

```bash
# 새 monorepo 시작
mkdir myorg && cd myorg && git init

# 각 repo 의 history 보존하며 가져오기
for repo in order-service payment-service user-service; do
  git clone https://github.com/myorg/$repo.git ../tmp-$repo
  cd ../tmp-$repo
  git filter-repo --to-subdirectory-filter services/$repo --force
  cd ../myorg
  git remote add $repo ../tmp-$repo
  git fetch $repo
  git merge --allow-unrelated-histories $repo/main
  git remote remove $repo
done
```

> **함정**: git history 가 거대해짐. 100+ repo 합치면 클론 수 GB. partial clone / sparse-checkout 활용.

### 7.2 Monorepo → 일부 Multi-repo 분리

`git filter-repo --path services/order --path-rename services/order:.` 로 부분 추출.

### 7.3 점진적 전환 (가장 현실적)

```
1. 현재 모든 새 서비스는 monorepo 에만 만든다
2. 기존 multi-repo 서비스는 그대로 유지
3. 다음 큰 리팩토링 / 재작성 시 monorepo 로 이전
4. 1년 뒤 multi-repo 가 외주/공개용만 남는다
```

빅뱅 마이그레이션은 거의 실패. **점진적**.

---

## 8. CI / 빌드 시간 — 가장 큰 함정

### 8.1 측정

```
naive (모두 빌드):       30분
incremental (변경만):    5분
remote cache (재사용):   1분
```

**naive monorepo = 안티패턴**. 처음부터 incremental 도구 필수.

### 8.2 incremental + remote cache 패턴

| 도구 | remote cache |
| --- | --- |
| Nx | Nx Cloud / S3 backend |
| Turborepo | Vercel Remote Cache / S3 |
| Bazel | Bazel Remote Cache (자체) |
| Gradle | Gradle Build Cache (자체) |

```yaml
# .github/workflows/ci.yml — Nx 예시
- uses: nrwl/nx-set-shas@v4
- run: pnpm nx affected --target=test --parallel=3
- run: pnpm nx affected --target=lint  --parallel=3
- run: pnpm nx affected --target=build --parallel=3
```

`affected --base=main --head=HEAD` 만 핵심.

---

## 9. 권한 / 코드 소유

### 9.1 Monorepo `CODEOWNERS`

```
# .github/CODEOWNERS
/services/order/        @myorg/order-team
/services/payment/      @myorg/payment-team
/libs/domain/order/     @myorg/order-team
/libs/shared/events/    @myorg/platform-team
/.github/workflows/     @myorg/platform-team
```

PR 이 `services/order/` 만 만지면 order-team 만 review 자동 지정.

### 9.2 Multi-repo

각 repo 의 GitHub Team 권한 + 브랜치 protection.

---

## 10. 안티패턴

### A1 — Monorepo + naive CI
모든 변경에 모든 테스트. 30분 PR. 점점 머지 회피 → 빅 PR.

### A2 — Multi-repo + "그냥 복사"
공유 라이브러리 만들 시간 아껴 코드 복사 → 6개월 뒤 7개 repo 에 같은 버그 7번 수정.

### A3 — Shared lib 무한 의존
공유 lib v1.2.3 의 변경 = 50개 서비스 빌드. **lib 는 framework 수준만**.

### A4 — Monorepo 안에 무한 패키지
30개 lib + 100개 서비스. 누가 무엇을 import 하는지 그래프가 spaghetti. **계층 enforce** (Nx `tags` / Bazel `visibility`).

```json
// Nx — order 만 import 가능한 lib 표시
{ "tags": ["scope:order", "type:domain"] }
// 그리고 .eslintrc 에 boundary rule
```

### A5 — Multi-repo + 의존성 drift
서비스 A 는 Spring Boot 2.7, B 는 3.0, C 는 3.3. 보안 패치 적용 = 50개 repo PR. **Renovate / Dependabot** + 표준 BOM.

### A6 — Monorepo 의 git log 무시
거대 monorepo 의 master log = 노이즈. 서비스별 view 도구 (`git log -- services/order/`) 또는 Sourcegraph.

---

## 11. 의사결정 트리

```
Q1. 팀 규모?
├─ ≤ 10명  → Monorepo (도구는 단순한 거: Turborepo / Gradle composite)
├─ 11~30명 → Monorepo + Nx 또는 Bazel
└─ 30명+   → 아래

Q2. 30명+ 에서 공유 도메인 코드가 많이 자주 변경되나?
├─ Yes → Monorepo + Bazel / Nx + remote cache
└─ No  → 

Q3. 외부 공개 / 외주 / 분리된 권한 필요?
├─ Yes → Hybrid (core monorepo + 외부 별 repo)
└─ No  → 둘 다 OK. 팀 자율성 우선이면 Multi.
```

---

## 12. 운영 체크리스트

### Monorepo
- [ ] 빌드 도구 (Nx / Bazel / Turborepo / Gradle) 결정 후 시작
- [ ] CI 에 `affected` 또는 `target = changes only` 적용
- [ ] Remote cache 도입 (Nx Cloud / Bazel cache / Gradle cache)
- [ ] `CODEOWNERS` 로 PR 라우팅
- [ ] partial clone / sparse-checkout 문서화 (대형 repo)
- [ ] `git lfs` (이미지 / 바이너리)
- [ ] 의존성 boundary enforcement (Nx tags / Bazel visibility)
- [ ] 새 서비스 / 라이브러리 generator
- [ ] 일관 lint / format 규칙
- [ ] 정기 빌드 시간 모니터링

### Multi-repo
- [ ] repo template (cookiecutter / Backstage scaffolder)
- [ ] 공유 라이브러리 SemVer + CHANGELOG 강제
- [ ] Renovate / Dependabot 표준
- [ ] 표준 CI workflow (재사용 가능한 workflow)
- [ ] 코드 검색 (Sourcegraph 또는 GitHub Enterprise)
- [ ] 보안 패치 / 라이브러리 업데이트 일괄 처리 절차
- [ ] 의존성 그래프 / 호출 관계 가시화 도구

---

## 13. 관련

- [[architecture|↑ architecture hub]]
- [[msa-overview]] — 배포 단위 결정 (직교 축)
- [[modular-monolith]] — monorepo 의 자연스러운 동거
- [[../../devops/devops|↗ devops]] — CI / CD
