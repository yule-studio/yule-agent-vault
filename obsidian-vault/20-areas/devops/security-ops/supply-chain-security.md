---
title: "Supply chain security — SBOM / cosign / SLSA / OIDC"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:10:00+09:00
tags: [devops, security-ops, supply-chain, sbom, cosign]
---

# Supply chain security — SBOM / cosign / SLSA / OIDC

**[[security-ops|↑ security-ops]]**

---

## 1. 왜

```
[소스] → [의존성] → [build] → [registry] → [deploy] → [prod]
   ↑        ↑          ↑         ↑          ↑          ↑
 작성    npm/Maven   CI server  image      ArgoCD    cluster
   │      compromise compromise compromise compromise compromise
   └────────────── 어느 단계도 공격 surface ─────────────┘
```

흔한 공격:
- **SolarWinds 2020** — build server 침투 → 18,000 고객.
- **Log4j 2021** — 의존성 RCE.
- **3CX 2023** — supply chain.
- **npm typosquatting** — `req3uests` 등 비슷 이름 악성.

---

## 2. SLSA framework (4 levels)

> Supply-chain Levels for Software Artifacts (Google)

| Level | 요구 |
| --- | --- |
| **L1** | build process 문서화, provenance 생성 |
| **L2** | 호스팅 build service (예: GitHub Actions), 서명 |
| **L3** | non-falsifiable provenance, 격리된 build |
| **L4** | 2-person review, hermetic build, parameterless |

---

## 3. SBOM (Software Bill of Materials)

```bash
# Syft
syft my-app:1.0 -o cyclonedx-json > sbom.json
syft my-app:1.0 -o spdx-json > sbom.spdx.json

# Trivy
trivy image --format cyclonedx --output sbom.json my-app:1.0
```

내용:
- 모든 의존성 (direct + transitive)
- 버전 / hash
- license
- supplier / vendor

→ CVE 발생 시 즉시 "어디 영향?" 파악 가능.

---

## 4. image signing (cosign ★)

```bash
# 설치
brew install cosign

# key 생성
cosign generate-key-pair

# 서명 (push 후)
cosign sign --key cosign.key registry.example.com/app:1.0

# 검증
cosign verify --key cosign.pub registry.example.com/app:1.0
```

→ image 가 진짜 우리 build 인지 검증.

---

## 5. keyless signing (★ OIDC + Fulcio)

```bash
# 회사 GitHub Actions 가 build → cosign + OIDC token
cosign sign --identity-token $TOKEN registry.example.com/app:1.0

# 검증 — 특정 identity 가 서명한 image 만 허용
cosign verify \
    --certificate-identity-regexp "^https://github.com/myorg/.*/.github/workflows/.*" \
    --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
    registry.example.com/app:1.0
```

→ key 관리 X. GitHub Actions OIDC token + Sigstore Fulcio CA.

→ **현재 표준**.

---

## 6. attestation (★)

서명 + 메타데이터:

```bash
# SBOM attestation
cosign attest --predicate sbom.json --type cyclonedx \
    --key cosign.key registry.example.com/app:1.0

# vuln scan attestation
cosign attest --predicate scan.json --type vuln \
    --key cosign.key registry.example.com/app:1.0

# 검증 — 모든 attestation
cosign verify-attestation --key cosign.pub registry.example.com/app:1.0
```

→ image 가 "scan 결과 통과 + SBOM 있음 + verified build" 모두 증명.

---

## 7. admission controller (k8s 차단)

```yaml
# Kyverno ClusterPolicy
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: {name: verify-images}
spec:
  validationFailureAction: enforce
  rules:
    - name: check-signature
      match: {resources: {kinds: [Pod]}}
      verifyImages:
        - imageReferences: ["registry.example.com/*"]
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/myorg/*"
                    issuer: "https://token.actions.githubusercontent.com"
```

→ 미서명 image 는 deploy 자체 거부.

대안: Sigstore Policy Controller, OPA Gatekeeper.

---

## 8. dependency pinning

```dockerfile
# ❌ tag 만
FROM nginx:latest         # latest 가 변경됨 → 다음 build 결과 달라짐

# ✅ digest (immutable)
FROM nginx:1.25.3-alpine@sha256:abcd1234...
```

```json
// package-lock.json / yarn.lock (commit)
// gradle 의 dependencyVerification
```

→ 의존성 hash 검증.

---

## 9. private registry / mirror

```
- Docker Hub rate limit (free: 200 pull/6h)
- 외부 registry 공격 = supply chain 침해

대안:
  - Harbor (OSS, scan + sign 내장)
  - AWS ECR / GCR / GHCR
  - Nexus / Artifactory
  - 회사 mirror (pull-through cache)
```

→ public image 도 회사 registry mirror 통과.

---

## 10. OIDC for CI (★ access key 없음)

```yaml
# GitHub Actions → AWS
permissions:
  id-token: write
  contents: read

jobs:
  build:
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123:role/github-actions-deploy
          aws-region: us-east-1
```

→ AWS Role 의 trust policy:
```json
{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::123:oidc-provider/token.actions.githubusercontent.com"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
        "StringEquals": {
            "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
        }
    }
}
```

→ 특정 repo의 main branch CI 만 deploy 가능. ★

---

## 11. CodeQL / SAST in CI

```yaml
- uses: github/codeql-action/init@v3
  with: {languages: java, kotlin}
- uses: github/codeql-action/analyze@v3
```

→ 코드의 보안 취약점 (XSS / SQLi / path traversal) 자동.

---

## 12. 함정

1. **Docker Hub rate limit** — production build CI 에 영향.
2. **`:latest` tag** — reproducible X.
3. **lockfile 안 commit** — 의존성 변경.
4. **build 환경 access** — 누구나 push.
5. **signing key 도난** — keyless (cosign + OIDC) 권장.
6. **admission controller 없음** — 어떤 image 든 cluster 에.
7. **3rd-party action 직접 reference (tag)** — `@v3` 가 변경됨. `@<sha>` 권장.

---

## 13. 관련

- [[security-ops|↑ security-ops]]
- [[vulnerability-management]]
- [[../docker/security|↗ docker security]]
- [[../cicd/cicd|↗ CI/CD]]
- [[iam-best-practices]]
