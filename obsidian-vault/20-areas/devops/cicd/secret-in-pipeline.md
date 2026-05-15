---
title: "Pipeline 의 secret 관리"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:38:00+09:00
tags: [devops, cicd, secret]
---

# Pipeline 의 secret 관리

**[[cicd|↑ cicd]]**

---

## 1. 어디 저장

| 위치 | 비교 |
| --- | --- |
| **CI secret store** (GitHub Secrets / GitLab Variables) | 단순 |
| **Vault** (HashiCorp) | 강 + dynamic |
| **AWS Secrets Manager / GCP SM** | cloud 통합 + IRSA |
| **SOPS + KMS** (file 암호화 git commit) | GitOps |
| **External Secrets Operator** (k8s) | Vault → k8s secret 동기 |

→ 시작: CI secret store. 성장: + Vault.

---

## 2. OIDC (cloud — keyless)

```yaml
# GitHub Actions → AWS
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123:role/github-deploy
      aws-region: ap-northeast-2
```

→ AK / SK 없이 IAM Role assume (단기 STS).

---

## 3. 함정

1. **secret echo / log** — 자동 mask but `printenv` / `set` 로 bypass 가능.
2. **fork PR 의 secret** — GitHub 가 자동 X.
3. **shell history** — `command < $SECRET_FILE`.
4. **artifact 에 secret 포함** — clean step.
5. **build log 의 stack trace** — secret 노출.

---

## 관련

- [[cicd|↑ cicd]]
- [[pipeline-patterns]]
- [[../security-ops/security-ops|↗ security-ops]]
