---
title: "이슈 템플릿 — 🚀 Release / Deployment"
kind: pattern
project: github-conventions
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:39:00+09:00
tags:
  - github
  - issue-template
  - release
  - deploy
---

# 이슈 템플릿 — 🚀 Release / Deployment

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 정리 |

**[[github-conventions|↑ GitHub 공통 작업 규칙]]**

---

## 1. 언제

새 버전 릴리즈 / 배포 시 체크리스트 추적.
- 정기 sprint 릴리즈
- hotfix 배포
- major / minor version 업그레이드

---

## 2. 라벨 / 메타

- **labels**: `🌏 Deploy`
- **assignees**: `codwithyc`
- **title prefix**: `[RELEASE] v` (예: `[RELEASE] v1.2.3`)

---

## 3. 템플릿 (`.github/ISSUE_TEMPLATE/release.md`)

```markdown
---
name: "🚀 Release / Deployment"
about: "새 버전 릴리즈 또는 배포 시 체크리스트용 템플릿입니다."
title: "[RELEASE] v"
labels: ["🌏 Deploy"]
assignees: [codwithyc]
---

## 📦 릴리즈 버전
> vX.Y.Z

## 📝 주요 변경사항
-

## 🔧 배포 전 체크리스트
- [ ] CI 파이프라인 성공
- [ ] 스테이징에서 기능 검증
- [ ] DB 마이그레이션 확인 (Flyway/Liquibase)
- [ ] 환경변수/시크릿 설정

## 🚀 배포 절차
1. `git tag vX.Y.Z && git push --tags`
2. 자동 배포 확인
3. Health check / Smoke test

## 🔄 롤백 가이드
> 이전 태그로 롤백하는 방법 등

## 📌 참고 링크
- CI 로그:
- 배포 문서: docs/DEPLOYMENT.md
```

---

## 4. 작성 가이드

### 4.1 SemVer (`vX.Y.Z`)

| 변경 | 증가 |
| --- | --- |
| breaking change (API 호환 X) | X (major) |
| 새 기능 호환 | Y (minor) |
| 버그 / 패치 | Z (patch) |

```
v1.2.3 → v2.0.0   breaking
v1.2.3 → v1.3.0   feature
v1.2.3 → v1.2.4   fix
```

### 4.2 주요 변경사항
- merged PR 목록 (`gh pr list --search "merged:>YYYY-MM-DD"`)
- CHANGELOG.md 의 [Unreleased] 섹션 옮기기
- 사용자가 보는 관점 (내부 리팩토링은 별도 섹션 또는 생략)

### 4.3 배포 전 체크리스트
프로젝트마다 추가:
- [ ] feature flag 상태 (활성 / 비활성 / 점진 ramp)
- [ ] 호환성 (이전 client 와)
- [ ] 모니터링 dashboard / alert 추가
- [ ] runbook 갱신
- [ ] 사용자 공지 (필요 시)

### 4.4 롤백 가이드
**모든 배포에 작성**. 30 분 안에 롤백 가능한가?
```
1. git tag vX.Y.Z-1 (이전 태그)
2. ArgoCD: sync to revision <옛 SHA>
3. DB migration 의 down() 검증
4. cache invalidation
```

---

## 5. priority_boost

`🌏 Deploy` = **+20**. 릴리즈는 다른 작업 차단할 수 있음 → 빠른 처리.

`+ 🏗 infrastructure` (+30) → 인프라 변경 동반 릴리즈.

---

## 6. 후속 작업

1. CI 통과 → tag push
2. GitHub Release 생성 (`gh release create vX.Y.Z`)
3. Slack / 메일 공지
4. Smoke test + monitoring 24h
5. 이슈 close + CHANGELOG.md commit

---

## 7. 관련

- [[github-conventions]] — hub
- [[pattern-issue-cicd]] — 파이프라인 자체
- [[pattern-label-rules]] — 🌏 Deploy
- [[pattern-workflow]] — 이슈 → PR → release
