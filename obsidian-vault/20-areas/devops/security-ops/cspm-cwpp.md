---
title: "CSPM / CWPP — cloud security posture / workload protection"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:59:00+09:00
tags: [devops, security-ops, cspm, cwpp, cnapp]
---

# CSPM / CWPP — cloud security posture / workload protection

**[[security-ops|↑ security-ops]]**

---

## 1. 용어 (★)

```
CSPM (Cloud Security Posture Management):
  cloud config 의 misconfiguration 탐지
  → 정적 (config / state)

CWPP (Cloud Workload Protection Platform):
  runtime workload 의 보안
  → 동적 (process / network / file)

CNAPP (Cloud-Native Application Protection Platform):
  CSPM + CWPP + 추가 (CI/CD / IaC / API / data) 통합
  → 2023+ 추세

CIEM (Cloud Infrastructure Entitlement Management):
  IAM permission analysis (least privilege)
```

→ 큰 회사 = CNAPP. 부분 = CSPM 또는 CWPP.

---

## 2. CSPM 의 흔한 발견

```
1. public S3 bucket
2. IAM root with active key
3. Security Group 0.0.0.0/0 to 22/3389
4. unencrypted RDS / EBS
5. CloudTrail disabled
6. EKS public endpoint
7. MFA 없는 admin
8. unused IAM key (90 days+)
9. inline policy (track 어려움)
10. expired cert
11. public AMI / snapshot
12. wide-open NACL
13. VPC Flow Logs off
14. wildcard certificate
15. cross-account trust 광범위
```

→ 매 발견 = 즉시 ticket.

---

## 3. CSPM 도구

| | type | 강점 |
| --- | --- | --- |
| **Wiz** (★) | SaaS | "agentless" 1등 |
| **Orca** | SaaS | snapshot 기반 |
| **Prisma Cloud** (Palo Alto) | SaaS | 종합 |
| **Lacework** | SaaS | ML 기반 |
| **CrowdStrike CSPM** | SaaS | EDR 통합 |
| **AWS Security Hub** | native | AWS only |
| **GCP Security Command Center** | native | GCP only |
| **Microsoft Defender for Cloud** | native | multi-cloud |
| **Prowler** (★ OSS) | CLI | scan |
| **ScoutSuite** (OSS) | CLI | scan |
| **CloudSploit / Aqua Security** | OSS+상용 | |

→ start = Prowler / Security Hub. enterprise = Wiz / Orca.

---

## 4. Prowler (★ OSS)

```bash
# install
pip install prowler

# AWS scan
prowler aws

# 특정 service / check
prowler aws --service s3
prowler aws --check check11   # specific check

# compliance framework
prowler aws --compliance cis_aws_v1.5
prowler aws --compliance soc2_aws

# 출력
prowler aws --output-formats html,json,csv
```

→ 매주 정기 + 결과 ticket.

---

## 5. AWS Security Hub

```bash
# 활성화
aws securityhub enable-security-hub

# standard 활성
aws securityhub batch-enable-standards \
    --standards-subscription-requests \
    StandardsArn=arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.4.0

# findings
aws securityhub get-findings \
    --filters '{"SeverityLabel":[{"Value":"CRITICAL","Comparison":"EQUALS"}]}'
```

자동 통합:
- GuardDuty (threat detection)
- Inspector (vuln scan)
- Macie (PII detection)
- Config (compliance)
- IAM Access Analyzer
- 3rd party (Wiz / Prisma 등)

---

## 6. AWS Config

```
config item:
  모든 자원의 config snapshot 시간별.

rule:
  expected state vs current
  - s3-bucket-public-read-prohibited
  - rds-storage-encrypted
  - iam-root-access-key-check
  - ...

remediation:
  자동 fix (옵션):
  - public S3 → block public 자동
  - root key → 비활성
```

→ AWS Config = compliance 의 backbone.

---

## 7. CWPP 핵심

```
runtime 보호 (container / VM 안):

1. process anomaly
   - shell in container
   - 새 binary execution
   - kernel module load

2. file integrity
   - /etc/passwd 변경
   - /usr/bin/ 새 file
   - SUID bit set

3. network anomaly
   - 새 outbound IP
   - C2 communication pattern
   - port scan

4. behavior baseline
   - 평소와 다른 syscall
   - resource 폭주
   - new user / group

5. exploit detection
   - kernel exploit
   - privilege escalation
```

---

## 8. CWPP 도구

| | type | 강점 |
| --- | --- | --- |
| **Falco** (★ OSS) | CNCF | k8s native |
| **Aqua Security** | 상용 | 종합 |
| **Sysdig Secure** | 상용 | Falco 기반 |
| **Wiz Runtime** | SaaS | CNAPP |
| **CrowdStrike Container** | EDR | endpoint 통합 |
| **Trend Cloud One** | 상용 | enterprise |
| **Twistlock / Prisma Compute** | 상용 | Palo Alto |
| **Tetragon** | OSS | eBPF (Cilium 팀) |
| **Tracee** (Aqua, OSS) | OSS | eBPF |

---

## 9. Falco 설치 + rule

```bash
helm install falco falcosecurity/falco \
    -n falco --create-namespace \
    --set tty=true \
    --set falcosidekick.enabled=true \
    --set falcosidekick.config.slack.webhookurl=https://hooks.slack.com/...
```

```yaml
# falco_rules.local.yaml (custom)
- rule: SSH into container
  desc: SSH login to running container
  condition: >
    spawned_process and proc.name = sshd and not container.image.repository in (allowed_ssh_images)
  output: "SSH process spawned (container=%container.name user=%user.name)"
  priority: WARNING
  tags: [container, ssh, mitre_lateral_movement]

- rule: Crypto miner activity
  desc: 의심 mining process
  condition: >
    spawned_process and 
    (proc.cmdline contains "xmrig" or proc.cmdline contains "monero")
  output: "Crypto miner: %proc.cmdline"
  priority: CRITICAL
```

→ output → Falcosidekick → Slack / PagerDuty / Loki.

---

## 10. Tetragon (★ Cilium 의 eBPF)

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata: {name: file-monitor}
spec:
  kprobes:
    - call: "fd_install"
      syscall: false
      args:
        - {index: 0, type: int}
        - {index: 1, type: "file"}
      selectors:
        - matchPIDs:
            - {operator: "NotIn", followForks: true, values: ["1"]}
          matchArgs:
            - {index: 1, operator: "Prefix", values: ["/etc/", "/root/"]}
          matchActions:
            - {action: Sigkill}    # kill process
```

→ eBPF 로 kernel 단 감지 + 즉시 kill 또는 alert.

---

## 11. CNAPP (★ 2024+ trend)

```
모든 layer 의 통합:
  
  Dev: SAST / SCA / IaC scan
  Build: SBOM / signing
  Deploy: admission / policy
  Runtime: CWPP / Falco
  Config: CSPM
  Network: 통신 visualizer
  Data: DSPM (data security)
  AI: AI-SPM
  Identity: CIEM
  
도구:
  - Wiz (★)
  - Orca
  - Palo Alto Prisma Cloud
  - Microsoft Defender for Cloud
  - Lacework
  - CrowdStrike

→ "all in one". 단 비싸.
```

---

## 12. CIEM (★ permission analysis)

```
IAM 의 over-permission 탐지:

method:
  - 실제 사용된 권한 추적 (CloudTrail)
  - 정의된 권한과 비교
  - 안 쓴 권한 → 권장 제거

도구:
  - AWS IAM Access Analyzer
  - GCP Policy Analyzer
  - Wiz CIEM
  - Sonrai
  - Ermetic
  - Authomize
```

```bash
# AWS IAM Access Analyzer
aws accessanalyzer create-analyzer \
    --analyzer-name my-analyzer \
    --type ACCOUNT

# findings
aws accessanalyzer list-findings --analyzer-arn ...

# unused permission
aws iam generate-organizations-access-report \
    --entity-path o-xxx
```

---

## 13. DSPM (Data Security Posture Management)

```
data 의 위치 / 종류 / access:
  - "어떤 S3 bucket 에 PII 있나?"
  - "누가 access 가능?"
  - "encryption 적용?"
  - "data classification"
  
도구:
  - AWS Macie
  - Cyera
  - BigID
  - Symmetry Systems
  - Dig Security
```

---

## 14. compliance mapping

```
CSPM 발견 → standard mapping:
  - CIS Benchmarks
  - NIST 800-53
  - PCI-DSS
  - HIPAA
  - SOC 2
  - ISO 27001
  - GDPR

→ audit 시 자동 evidence.
```

---

## 15. agent vs agentless

```
agent based (전통):
  pros: 깊은 visibility
  cons: 운영 부담 / 호환성

agentless (★ Wiz / Orca):
  pros: 빠른 적용 / 모든 자원
  cons: 깊이 약간 ↓ / API 의존

→ 새 도구 = agentless 가 표준.
```

---

## 16. 운영 (★)

```
☐ daily: critical finding review
☐ weekly: high / med review + ticket
☐ monthly: compliance report
☐ quarterly: tuning (false positive)

KPI:
  - mean time to remediate
  - critical finding open (target < 5)
  - compliance score
  - drift event count
```

---

## 17. 함정

1. **CSPM 만, runtime 무관심** — config OK 후 침해.
2. **alert noise** — 같은 finding 매일.
3. **자동 remediation 의 사고** — read-only 부터.
4. **agentless 의 false sense** — 일부 visibility 약함.
5. **compliance score 만** — 실제 보안 ≠ score.
6. **multi-cloud 한 도구 만** — coverage gap.
7. **CIEM 안 함** — over-permission 누적.

---

## 18. 관련

- [[security-ops|↑ security-ops]]
- [[siem]]
- [[devsecops]]
- [[threat-hunting]]
- [[compliance]]
