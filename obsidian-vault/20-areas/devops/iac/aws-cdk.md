---
title: "AWS CDK"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:12:00+09:00
tags: [devops, iac, aws, cdk]
---

# AWS CDK

**[[iac|↑ iac]]**

---

## 1. 무엇

- AWS 전용 IaC (TS / Python / Java / .NET / Go).
- 내부적으로 CloudFormation 생성.
- L1 (Cfn) / L2 (개선) / L3 (pattern) construct.

---

## 2. 시작 (TypeScript)

```bash
npm install -g aws-cdk
mkdir myapp && cd myapp
cdk init app --language typescript
```

---

## 3. 예시

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as rds from 'aws-cdk-lib/aws-rds';

export class ProdStack extends cdk.Stack {
    constructor(scope: cdk.App, id: string) {
        super(scope, id);

        const vpc = new ec2.Vpc(this, 'Vpc', {
            maxAzs: 2,
            cidr: '10.0.0.0/16',
        });

        const db = new rds.DatabaseInstance(this, 'Db', {
            engine: rds.DatabaseInstanceEngine.postgres({
                version: rds.PostgresEngineVersion.VER_16,
            }),
            instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MICRO),
            vpc,
        });
    }
}

const app = new cdk.App();
new ProdStack(app, 'ProdStack');
```

```bash
cdk synth        # CloudFormation YAML 출력
cdk diff
cdk deploy
cdk destroy
```

---

## 4. construct level

- **L1** (Cfn) — CloudFormation 1:1 — `CfnBucket`.
- **L2** — 개선 (default + helper) — `Bucket`.
- **L3 (Pattern)** — 다중 자원 패턴 — `ApplicationLoadBalancedFargateService`.

```typescript
import { ApplicationLoadBalancedFargateService } from 'aws-cdk-lib/aws-ecs-patterns';

new ApplicationLoadBalancedFargateService(this, 'Service', {
    cluster, taskImageOptions: {...}, publicLoadBalancer: true,
});
// 자동 — Fargate + ALB + Service + Auto-scaling
```

---

## 5. CDK vs Terraform

| | CDK | Terraform |
| --- | --- | --- |
| Cloud | AWS only (CDKtf 별도) | 모든 cloud |
| 언어 | TS/Py/Java/.NET | HCL |
| state | CFN stack | external |
| 학습 | AWS depth ↑ | 표준 |
| 본 vault | AWS only 면 | 시작 |

---

## 6. 관련

- [[iac|↑ iac]]
- [[tools-comparison]]
- [[../cloud-aws/cloud-aws|↗ AWS]]
