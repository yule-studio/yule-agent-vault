---
title: "DevSecOps — shift-left security pipeline ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:55:00+09:00
tags: [devops, security-ops, devsecops, shift-left]
---

# DevSecOps — shift-left security pipeline ★

**[[security-ops|↑ security-ops]]**

---

## 1. 개념

```
전통:
  코드 → build → deploy → security review (마지막)
  → 사고 발견 늦음, fix 비쌈

DevSecOps:
  보안을 모든 단계에 (shift-left):
    IDE → commit → CI → CD → runtime
  
→ "security as code" / "security as a developer feature".
```

---

## 2. 8 단계 통합

```
1. plan / design  → threat modeling
2. IDE             → linter / SAST plugin
3. commit          → pre-commit hook / secret scan
4. PR              → automated review / CodeQL
5. CI / build      → SCA / SAST / IaC scan / image scan
6. registry        → signing / SBOM / attestation
7. deploy          → admission control / policy
8. runtime         → IDS / Falco / behavior monitoring
```

---

## 3. IDE / pre-commit (★)

```bash
# pre-commit framework
pip install pre-commit

# .pre-commit-config.yaml
repos:
  - repo: https://github.com/zricethezav/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks               # secret detection
  - repo: https://github.com/trufflesecurity/trufflehog
    rev: v3.0.0
    hooks:
      - id: trufflehog
        args: ['--regex', '--entropy=False']
  - repo: https://github.com/bridgecrewio/checkov
    rev: 3.0.0
    hooks:
      - id: checkov                # IaC security
  - repo: local
    hooks:
      - id: detect-secrets
        name: detect-secrets
        entry: detect-secrets-hook
        language: python
        types: [text]

pre-commit install      # one-time
```

→ commit 전에 자동 scan. local 에서 즉시 feedback.

---

## 4. CI pipeline (★ 표준)

```yaml
# GitHub Actions 예
name: security
on: [push, pull_request]

jobs:
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: {fetch-depth: 0}
      - uses: gitleaks/gitleaks-action@v2

  sast:
    runs-on: ubuntu-latest
    permissions: {security-events: write}
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with: {languages: java, kotlin}
      - uses: github/codeql-action/analyze@v3
      
      # Semgrep (★ 더 빠름)
      - uses: returntocorp/semgrep-action@v1
        with:
          config: 'p/security-audit p/owasp-top-ten'

  sca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: snyk/actions/setup@master
      - run: snyk test
      
      # 또는 Dependabot (자동)

  iac-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: bridgecrewio/checkov-action@master
        with: {framework: terraform,kubernetes,dockerfile}
      - uses: aquasecurity/trivy-action@master
        with: {scan-type: config, scan-ref: .}

  image-scan:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
          ignore-unfixed: true

  sign:
    runs-on: ubuntu-latest
    needs: image-scan
    permissions: {id-token: write, packages: write}
    steps:
      - name: cosign sign
        run: |
          cosign sign --yes \
            ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
```

---

## 5. SAST (Static Application Security Testing)

| | type | 강점 |
| --- | --- | --- |
| **CodeQL** (GitHub) | OSS | Security 탭 통합 |
| **Semgrep** | OSS+SaaS | 빠름, custom rule 쉬움 |
| **SonarQube** | OSS+상용 | 종합 quality + 보안 |
| **Checkmarx** | enterprise | 큰 회사 |
| **Veracode** | enterprise | regulator |
| **Snyk Code** | SaaS | 통합 platform |
| **Fortify** | enterprise | legacy |

→ OSS = CodeQL / Semgrep. enterprise = Snyk / Checkmarx.

---

## 6. SCA (Software Composition Analysis)

```
의존성의 known CVE:
  - npm / Maven / pip / Go 등의 library
  - direct + transitive 모두
  
도구:
  - Dependabot (★ GitHub native)
  - Snyk
  - OWASP Dependency-Check
  - Sonatype Nexus IQ
  - GitHub Advisory Database
  - Tidelift
```

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: gradle
    directory: /
    schedule: {interval: weekly}
    open-pull-requests-limit: 10
    labels: [dependencies, security]
    groups:
      security-updates:
        update-types: [security]      # 보안 패치만 즉시 PR
```

---

## 7. IaC scan

```bash
# Checkov (★)
checkov -d . --framework terraform,kubernetes,dockerfile,helm

# tfsec
tfsec .

# kubesec
kubesec scan deployment.yaml

# Terrascan
terrascan scan -i terraform

# OPA / Conftest (custom policy)
conftest test --policy policies/ infra/
```

```rego
# policies/k8s.rego
package main

deny[msg] {
    input.kind == "Pod"
    input.spec.containers[_].securityContext.privileged == true
    msg := "privileged container 금지"
}

deny[msg] {
    input.kind == "Pod"
    not input.spec.containers[_].resources.limits
    msg := "resource limit 필수"
}
```

---

## 8. DAST (Dynamic Application Security Testing)

```
running app 의 fuzz / attack 시도:

도구:
  - OWASP ZAP (OSS)
  - Burp Suite (상용 표준)
  - Acunetix
  - Invicti
  - Rapid7 InsightAppSec
```

```yaml
# ZAP in CI (staging 대상)
- uses: zaproxy/action-baseline@v0.10.0
  with:
    target: 'https://staging.example.com'
    rules_file_name: '.zap/rules.tsv'
    cmd_options: '-a'
```

---

## 9. secret scanning

```bash
# Gitleaks (★)
gitleaks detect --source . --verbose

# TruffleHog
trufflehog git file://.

# detect-secrets (Yelp)
detect-secrets scan > .secrets.baseline
detect-secrets audit .secrets.baseline

# GitHub secret scanning (free for public, paid for private)
# 자동 활성

# AWS / GCP / Azure 등 token 자동 인식
```

→ commit 전 + repo 전체 history 둘 다.

→ 발견 시: rotate immediately + git filter-repo.

---

## 10. admission control (★ deploy)

```yaml
# OPA Gatekeeper
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata: {name: k8spsprequiredsecuritycontext}
spec:
  crd:
    spec:
      names: {kind: K8sPSPRequiredSecurityContext}
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spspsecuritycontext
        violation[{"msg": msg}] {
          c := input.review.object.spec.containers[_]
          not c.securityContext.runAsNonRoot
          msg := sprintf("Container %v must runAsNonRoot", [c.name])
        }
```

```yaml
# Kyverno (★ 더 쉬움)
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: {name: require-image-signing}
spec:
  validationFailureAction: enforce
  rules:
    - name: verify-images
      match: {resources: {kinds: [Pod]}}
      verifyImages:
        - imageReferences: ["registry.example.com/*"]
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/myorg/*/.github/workflows/*"
                    issuer: "https://token.actions.githubusercontent.com"
```

---

## 11. CSPM (Cloud Security Posture Management)

```
cloud config 의 misconfiguration 탐지:

도구:
  - AWS Security Hub
  - Prowler (OSS)
  - ScoutSuite (OSS)
  - Wiz / Orca (SaaS)
  - Lacework
  - Palo Alto Prisma
  - Trend Micro

발견:
  - S3 public
  - IAM root access key
  - Security Group 0.0.0.0/0
  - encryption 안 됨
  - CloudTrail off
  - unencrypted RDS
  - public snapshot
```

---

## 12. CWPP (Cloud Workload Protection Platform)

```
runtime workload 보호:

도구:
  - Falco (★ OSS)
  - Aqua Security
  - Sysdig
  - Wiz Runtime
  - Trend Micro Deep Security
  - CrowdStrike Falcon
  - Twistlock (Palo Alto)

기능:
  - container runtime behavior
  - file integrity
  - network anomaly
  - process anomaly
  - kernel exploit detection
```

---

## 13. supply chain (★ 2023+)

```
SLSA framework (Google):
  L1: build process documented
  L2: hosted build + signed provenance
  L3: hardened build + non-falsifiable
  L4: 2-person review + hermetic

practical:
  ☐ SBOM 생성 (Syft / Trivy)
  ☐ image signing (cosign)
  ☐ keyless signing (Sigstore + OIDC)
  ☐ provenance attestation
  ☐ admission control (signed images only)
  ☐ dependency pinning (digest, lockfile)
  ☐ private mirror (no direct Docker Hub)
```

→ [[supply-chain-security]] 참조.

---

## 14. policy as code

```
all security policy = git commit + PR review.

도구:
  - OPA / Gatekeeper
  - Kyverno
  - Sentinel (Terraform Cloud)
  - Pulumi CrossGuard
  - Checkov (custom)
```

```rego
# policies/aws/s3.rego
package terraform.aws.s3

deny[msg] {
    resource := input.resource.aws_s3_bucket[name]
    not resource.acl == "private"
    msg := sprintf("S3 bucket %v must be private", [name])
}

deny[msg] {
    resource := input.resource.aws_s3_bucket[name]
    not contains_encryption(resource)
    msg := sprintf("S3 bucket %v must have encryption", [name])
}
```

---

## 15. KPI (★ shift-left 측정)

```
- mean time to detect (MTTD)        — incident detect 시간
- mean time to remediate (MTTR)
- vulnerability backlog              — open CVE
- vulnerability fix time              — Critical < 24h?
- security tests in CI               — coverage
- secrets in commits                  — should be 0
- failed deployments due to security
- mean age of known vulnerability     — patch 빈도
- 직원 security training 완료율
- pentest finding closure rate
```

---

## 16. team / culture

```
DevSecOps:
  - 보안 = 모두의 책임
  - security champion in each team
  - threat modeling 정기
  - bug bounty
  - red team exercise
  - blameless postmortem

vs traditional:
  - security team 따로
  - "보안이 막아"
  - 사고 시 책임 추궁
```

---

## 17. 함정

1. **tool 만, culture 없음** — alert fatigue.
2. **noise (false positive) 무시 안 함** — 진짜 위협 묻힘.
3. **CI fail 우회** — `--skip` 남발.
4. **shift-left 만, runtime 무관심** — 0-day.
5. **policy 너무 엄격** — 우회 / fork 위험.
6. **secret 발견 시 단순 fix** — rotate + history 정리 필요.
7. **regulatory 의무 무시** — fine.

---

## 18. 관련

- [[security-ops|↑ security-ops]]
- [[vulnerability-management]]
- [[supply-chain-security]]
- [[threat-hunting]]
- [[../cicd/pipeline-patterns|↗ CI/CD]]
