---
title: "Release strategies — semver / tag / changelog"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:44:00+09:00
tags: [devops, cicd, release]
---

# Release strategies — semver / tag / changelog

**[[cicd|↑ cicd]]**

---

## 1. Semver

```
1.2.3
│ │ └─ Patch (버그 수정, backward-compat)
│ └─── Minor (기능 추가, backward-compat)
└───── Major (breaking change)
```

---

## 2. Tag-based release

```bash
git tag v1.2.0
git push origin v1.2.0
```

```yaml
# GitHub Actions
on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
```

---

## 3. Conventional Commits + semantic-release

```
feat: new feature       → minor up
fix: bug                → patch up
feat!: breaking         → major up
chore / docs / refactor → no bump
```

→ `semantic-release` 가 자동 tag + changelog + image tag.

---

## 4. CHANGELOG.md (자동 생성)

```
# Changelog

## [1.2.0] - 2026-05-15
### Added
- 회원가입 OAuth2

### Fixed
- 결제 race condition
```

→ `git-cliff` / `standard-version` / `release-please` 같은 도구.

---

## 5. Pre-release / beta

```
v1.2.0-beta.1
v1.2.0-rc.1
v1.2.0
```

---

## 6. 함정

1. **manual tag** → 일관성 X.
2. **semver 위반** (minor 라고 표시 + breaking change) → 사용자 fail.
3. **changelog 누락** → release note 의미 X.
4. **registry retention** — 옛 tag 누적 → cleanup.

---

## 관련

- [[cicd|↑ cicd]]
- [[pipeline-patterns]]
- [[../docker/registry|↗ registry tag 전략]]
