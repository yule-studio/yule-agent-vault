---
title: "Pulumi — typed IaC"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:10:00+09:00
tags: [devops, iac, pulumi]
---

# Pulumi — typed IaC

**[[iac|↑ iac]]**

---

## 1. 무엇

- TS / Python / Go / .NET 같은 real language 로 인프라.
- IDE autocomplete + type check + test.
- state = Pulumi Service (default) or S3 / Azure / GCS.

---

## 2. 설치 + 시작

```bash
brew install pulumi
pulumi new typescript-aws
```

---

## 3. 예시 (TypeScript + AWS)

```typescript
import * as aws from "@pulumi/aws";

const vpc = new aws.ec2.Vpc("main", {
    cidrBlock: "10.0.0.0/16",
    tags: { Name: "prod-vpc" },
});

const subnet = new aws.ec2.Subnet("public", {
    vpcId: vpc.id,
    cidrBlock: "10.0.1.0/24",
    availabilityZone: "ap-northeast-2a",
});

const ami = aws.ec2.getAmi({
    mostRecent: true,
    owners: ["amazon"],
    filters: [{ name: "name", values: ["amzn2-ami-hvm-*-x86_64-gp2"] }],
});

const instance = new aws.ec2.Instance("web", {
    ami: ami.then(a => a.id),
    instanceType: "t3.micro",
    subnetId: subnet.id,
});

export const ip = instance.publicIp;
```

```bash
pulumi up
pulumi destroy
pulumi stack output ip
```

---

## 4. Pulumi 의 강점

- ✓ IDE 의 모든 기능 (autocomplete / refactor / find usage).
- ✓ Test (jest / pytest) 로 검증.
- ✓ loop / if / class — real language.
- ✓ Cross-stack reference 강.

---

## 5. Pulumi vs Terraform

| | Terraform | Pulumi |
| --- | --- | --- |
| 언어 | HCL | TS/Py/Go/.NET |
| state | external | Pulumi Service or self |
| 학습 | 쉬움 | 약간 ↑ |
| 자료 | 많음 | 적음 |
| 본 vault | ★ (시작) | typed 필수 시 |

---

## 6. 관련

- [[iac|↑ iac]]
- [[terraform-basics]]
- [[tools-comparison]]
