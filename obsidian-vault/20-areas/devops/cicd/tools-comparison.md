---
title: "CI/CD 도구 비교 ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:17:00+09:00
tags: [devops, cicd, comparison]
---

# CI/CD 도구 비교 ★

**[[cicd|↑ cicd]]**

---

## 1. 매트릭스

| 도구 | 호스팅 | 가격 | 학습 | 통합 | 특징 |
| --- | --- | --- | --- | --- | --- |
| **GitHub Actions** ★ | SaaS + self-hosted runner | Free (public/한도) / $ | 쉬움 | GitHub | 가장 인기, marketplace |
| **GitLab CI** | SaaS + self-hosted | Free (한도) / $ | 쉬움 | GitLab | DevOps platform 통합 |
| **Jenkins** | self-hosted | OSS (인프라 비용) | 어려움 | plugin 수천 | enterprise 표준 |
| **CircleCI** | SaaS | Free (한도) / $ | 쉬움 | 모든 git | orb (plugin) |
| **Buildkite** | hybrid (UI SaaS + self agent) | $ | 중간 | self-hosted runner | 분산 agent |
| **Tekton** | k8s 네이티브 | OSS | 어려움 | k8s | CRD 기반 |
| **Drone** | self-hosted | OSS | 쉬움 | git | 가벼움 |
| **Argo Workflows** | k8s | OSS | 중간 | k8s | DAG |
| **TeamCity** | hybrid | $$ (Free 한도) | 중간 | 모든 git | JetBrains |
| **AWS CodePipeline** | SaaS | $ | 중간 | AWS | AWS 통합 |
| **GCP Cloud Build** | SaaS | $ | 쉬움 | GCP | GCP 통합 |

---

## 2. CD 도구 (배포 자동)

| | GitOps | k8s | UI |
| --- | --- | --- | --- |
| **ArgoCD** ★ | O | O | 강 |
| **Flux** | O | O | X (CLI) |
| **Argo Rollouts** | + canary/blue-green | O | O |
| **Spinnaker** | weak | O | 강 (Netflix) |
| **Octopus Deploy** | X | O + VM | 강 (.NET 강세) |

---

## 3. 본 vault 권장 조합

| 상황 | CI | CD |
| --- | --- | --- |
| **시작** | GitHub Actions | ssh / kubectl apply |
| **k8s 도입 후** | GitHub Actions | ArgoCD (GitOps) |
| **enterprise** | GitLab CI | ArgoCD |
| **self-hosted / OSS only** | Jenkins | ArgoCD |
| **분산 monorepo** | Buildkite | ArgoCD |
| **k8s native** | Tekton | ArgoCD |
| **AWS only** | CodePipeline | CodeDeploy |
| **GCP only** | Cloud Build | Cloud Deploy |

---

## 4. 도구별 강점 / 약점

### GitHub Actions

- ✓ git push 만 하면 동작
- ✓ marketplace (actions 수천 개)
- ✓ self-hosted runner (보안)
- ✗ secret 의 cross-repo 공유 제한
- ✗ 큰 monorepo 시 느림 (path filter 필요)

### GitLab CI

- ✓ DevOps platform 통합 (issue / wiki / registry)
- ✓ self-hosted free
- ✓ pipeline graph UI 강
- ✗ GitLab 외 git 사용 시 제한

### Jenkins

- ✓ 플러그인 수천 개 — 무한 customize
- ✓ enterprise 표준 (자체 운영)
- ✗ 학습 곡선 매우 ↑
- ✗ Jenkinsfile groovy 복잡
- ✗ 자체 운영 부담

### ArgoCD

- ✓ GitOps 표준
- ✓ k8s native
- ✓ UI 강
- ✓ multi-cluster
- ✗ k8s 외 deploy X

### Tekton

- ✓ k8s native (CRD)
- ✓ reusable Task
- ✗ 학습 어려움 (DSL 없음, YAML 만)
- ✗ UI 부족 (Dashboard 별도)

---

## 5. 본 vault 의 의사결정

```
GitHub repo? YES → GitHub Actions
GitHub repo? NO + GitLab? → GitLab CI
self-hosted only? → Jenkins (legacy) or GitLab CI (modern)
k8s deploy? → + ArgoCD
canary / progressive delivery? → + Argo Rollouts / Flagger
```

---

## 6. 관련

- [[cicd|↑ cicd]]
- [[github-actions]]
- [[gitlab-ci]]
- [[jenkins]]
- [[argocd-cd]]
