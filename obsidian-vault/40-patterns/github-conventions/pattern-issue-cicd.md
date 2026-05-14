---
title: "이슈 템플릿 — 🛠 CI/CD Config Request"
kind: pattern
project: github-conventions
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:37:00+09:00
tags:
  - github
  - issue-template
  - cicd
  - deploy
---

# 이슈 템플릿 — 🛠 CI/CD Config Request

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 정리 |

**[[github-conventions|↑ GitHub 공통 작업 규칙]]**

---

## 1. 언제

GitHub Actions / Jenkins / 파이프라인 추가 / 변경 요청.
- 새 워크플로우 추가
- 트리거 조건 변경
- 배포 자동화
- 시크릿 / 환경 변수 관리

---

## 2. 라벨 / 메타

- **labels**: `🌏 Deploy`, `help wanted` (옵션)
- **assignees**: `codwithyc`
- **title prefix**: `[CI/CD] `

---

## 3. 템플릿 (`.github/ISSUE_TEMPLATE/cicd_config.md`)

```markdown
---
name: "🛠 [CI/CD] Config Request"
about: "GitHub Actions 등 CI/CD 워크플로우 추가/변경 이슈 템플릿"
title: "[CI/CD] "
labels: ["🌏 Deploy", "help wanted"]
assignees: [codwithyc]
---

## 🔨 워크플로우 목적
> 예) develop 머지 시 스테이징 배포 자동화

## 🔑 트리거
- 브랜치 / 태그:

## 📜 수행 스텝
1. Build
2. Test
3. Docker 이미지 빌드 & 푸시
4. 배포

## 🚨 주의 사항 / 고려 사항
> 권한·시크릿 관리, 롤백 전략 등
```

---

## 4. 작성 가이드

### 4.1 워크플로우 목적
**한 문장** + **트리거 시점 + 결과**.
```
✅ "develop 브랜치 머지 시 staging 환경에 자동 배포 + Slack 알림"
❌ "배포 자동화"
```

### 4.2 트리거
GitHub Actions 의 `on:` 부분 명시.
```
- push / branches: [develop]
- pull_request / branches: [main]
- workflow_dispatch
- schedule (cron)
- release
```

### 4.3 수행 스텝
의존성 / 순서 / 병렬 가능성 명시.
```
1. checkout
2. setup Java 21
3. cache (gradle / maven)
4. build + test (병렬 가능)
5. SonarCloud
6. Docker build + push (test 통과 시)
7. ArgoCD sync (manual approve)
```

### 4.4 주의 사항
- **시크릿** — repo secret vs environment secret
- **권한** — `permissions:` 최소 권한 (token theft 방어)
- **롤백** — 실패 시 이전 image / tag 로 복귀
- **비용** — runner 시간 / Docker registry pull rate
- **보안** — `pull_request_target` 의 위험

---

## 5. priority_boost

`🌏 Deploy` = **+20**. 인프라 / 도구 관련은 보통 우선 처리 (다른 작업 차단).

`+ 🏗 infrastructure` (+30) → CI/CD 인프라 자체 변경.

---

## 6. 후속 작업

1. dev 브랜치에서 워크플로우 테스트 (`act` 또는 dev branch + ghcr)
2. PR 에 dry-run / staging 결과 첨부
3. main 머지 후 production 트리거 검증
4. README / `docs/CI_CD.md` 업데이트

---

## 7. 관련

- [[github-conventions]] — hub
- [[../../20-areas/devops/github-actions/github-actions]] — Actions 패턴 / syntax
- [[pattern-issue-release]] — 배포 체크리스트
- [[pattern-label-rules]]
