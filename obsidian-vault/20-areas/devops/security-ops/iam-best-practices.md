---
title: "IAM best practices — least privilege"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:04:00+09:00
tags: [devops, security-ops, iam]
---

# IAM best practices — least privilege

**[[security-ops|↑ security-ops]]**

---

## 1. 원칙

1. **least privilege** — 최소 권한.
2. **deny by default** — 기본 거부, 명시 허용.
3. **separation of duties** — 다른 사람.
4. **role > user 직접 policy** — 재사용 + audit.
5. **MFA 필수** — 특히 root / admin.
6. **rotation** — credential 정기 변경.
7. **no root for daily** — root 는 처음 setup 만.
8. **federate** — SSO (Okta / Azure AD) + IAM Identity Center.
9. **audit log** — CloudTrail / Cloud Audit Log.
10. **time-limited** — session 24h 등.

---

## 2. AWS IAM 구조

```
User                — 개인 / app
Group               — User 묶음 + policy
Role                — 임시 권한 (assume)
Policy              — 권한 (JSON)
Service-linked Role — service 가 사용
```

---

## 3. Policy 예 (least privilege)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::my-bucket/uploads/*"
        }
    ]
}
```

→ `*` (모든 권한) 피하고 명시.

❌ 나쁨:
```json
{ "Action": "s3:*", "Resource": "*" }     // 모든 S3 모든 bucket
```

---

## 4. role 사용 (★)

### EC2 → S3

```yaml
# IAM Role: ec2-s3-role
# Trust policy: ec2.amazonaws.com 가 assume 가능
# attached: s3 read-only policy

# EC2 instance 에 attach
# → SDK 가 자동 credential (메타데이터 service)
```

→ EC2 안에 access key 둘 필요 X. ★

### Lambda

```yaml
# Lambda 에 Role 부여
# function 코드 안에 credential 없음
```

### k8s Pod → AWS (IRSA / Workload Identity)

```yaml
# AWS — IRSA (IAM Roles for Service Accounts)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123:role/app-role

---
spec:
  serviceAccountName: app-sa
  # → pod 안의 SDK 가 자동 IAM Role
```

→ k8s pod 에 access key 둘 필요 X.

### CI/CD → AWS (OIDC)

```yaml
# GitHub Actions → AWS (key 없이)
permissions:
  id-token: write       # OIDC

jobs:
  deploy:
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123:role/github-actions
          aws-region: ap-northeast-2
```

→ GitHub 의 OIDC token 으로 AWS Role assume. **access key 영구 저장 X**. ★

---

## 5. RBAC vs ABAC

```
RBAC (Role-Based):     User → Role → Permission
                       단순, 표준
                       
ABAC (Attribute-Based): User attributes + Resource attributes + Context → 결정
                        세밀하지만 복잡 (AWS Tag-based, IAM condition)
```

```json
// ABAC 예 — tag 가 같으면 허용
{
    "Effect": "Allow",
    "Action": "ec2:*",
    "Resource": "*",
    "Condition": {
        "StringEquals": {
            "ec2:ResourceTag/team": "${aws:PrincipalTag/team}"
        }
    }
}
```

---

## 6. SSO / 연합 (federate)

```
[사용자]
   ↓ Okta / Azure AD (회사 계정)
[IdP (SAML / OIDC)]
   ↓ token
[AWS IAM Identity Center]
   ↓ assume Role
[AWS account 들]
```

→ 입사 / 퇴사 시 IdP 만 관리. AWS 자체 User 안 만듦.

---

## 7. MFA 강제

```json
// IAM policy 에 MFA 없으면 deny
{
    "Effect": "Deny",
    "Action": "*",
    "Resource": "*",
    "Condition": {
        "BoolIfExists": { "aws:MultiFactorAuthPresent": "false" }
    }
}
```

→ root + 모든 IAM User MFA.

---

## 8. CloudTrail (audit log)

```
모든 AWS API 호출 → CloudTrail → S3
who: alice
when: 2026-05-15T08:00:00
what: s3:DeleteBucket on my-bucket
ip: 1.2.3.4
mfa: true
```

→ 모든 account 의 CloudTrail 켜기 (organization-wide).

---

## 9. permission boundary (★)

```
Role A 의 권한 = (attached policy) ∩ (permission boundary)
```

→ admin 이 만든 Role 이라도 boundary 이상 못 함.  
→ 개발자에게 IAM 권한 주되 escalation 막을 때.

---

## 10. service control policy (SCP, organization)

```
Organization 단위로 강제:
  - region 제한 (ap-northeast-2 / us-east-1 만)
  - service 제한 (특정 service 금지)
  - root user 제한
```

```json
{
    "Effect": "Deny",
    "Action": "*",
    "Resource": "*",
    "Condition": {
        "StringNotEquals": {
            "aws:RequestedRegion": ["ap-northeast-2", "us-east-1"]
        }
    }
}
```

→ 실수로 EU region 에 자원 만드는 것 차단.

---

## 11. 흔한 패턴

```
prod-account:        production 만
dev-account:         개발
shared-services:     CI, Vault, log central
audit-account:       CloudTrail / GuardDuty 모음
sandbox-*:           개발자 개인 (free play)
```

→ multi-account 가 표준. organization 으로 묶음.

---

## 12. k8s RBAC

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {namespace: prod, name: pod-reader}
rules:
  - apiGroups: [""]
    resources: [pods]
    verbs: [get, list, watch]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {namespace: prod, name: alice-pod-reader}
subjects:
  - kind: User
    name: alice
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

→ namespace 별 권한. ClusterRole / ClusterRoleBinding 은 cluster-wide.

---

## 13. 함정

1. **`*` 모든 action / resource** — least privilege 위반.
2. **access key 코드에 hardcode** — IAM Role 사용.
3. **root credential 일상 사용** — MFA + 가장자리 보관.
4. **권한 너무 늘림 → 줄이기 어려움** — 만들 때 최소.
5. **퇴사 시 access key 제거 안 함** — 즉시 disable.
6. **CloudTrail 안 켬** — audit 0.
7. **production / dev 같은 account** — blast radius ↑.
8. **shared admin account** — 누가 했는지 모름.

---

## 14. 관련

- [[security-ops|↑ security-ops]]
- [[zero-trust]]
- [[../kubernetes/rbac|↗ k8s RBAC]]
- [[../cloud-aws/cloud-aws|↗ AWS]]
