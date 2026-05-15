---
title: "SIEM — Splunk / Elastic Security / Wazuh / Datadog"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:12:00+09:00
tags: [devops, security-ops, siem]
---

# SIEM — Splunk / Elastic Security / Wazuh / Datadog

**[[security-ops|↑ security-ops]]**

---

## 1. SIEM 이란

**Security Information and Event Management**:
- 모든 source 의 log + event 집계
- 상관 분석 (correlation rule)
- alert (이상 행동)
- 검색 / 조사 (forensics)
- compliance report

→ "log 모이는 곳 + 보안 시각" 의 monitoring (SOC 의 핵심 tool).

---

## 2. 일반 monitoring 과 차이

| | Observability | SIEM |
| --- | --- | --- |
| 목적 | 시스템 건강 / 성능 | 보안 / 침해 |
| log | app / infra | + auth, network, audit |
| alert | error rate 등 | 침해 패턴 |
| retention | 7-30일 | 1년+ (regulator) |
| 검색 | 짧은 window | 긴 history |
| 도구 | Prometheus / Grafana | Splunk / Elastic Security |

→ 보통 같은 stack (ELK) 위에 다른 dashboard / rule.

---

## 3. 수집 source

```
Application:
  - access log (auth 성공/실패)
  - error log
  - audit log (관리자 action)

OS / infra:
  - auth log (sshd, sudo)
  - syslog
  - container runtime
  - k8s audit log

Cloud:
  - AWS CloudTrail (모든 API)
  - GCP Cloud Audit Log
  - VPC Flow Logs
  - Route53 query log

Security:
  - WAF log
  - IDS / IPS (Suricata, Snort)
  - EDR / antivirus
  - VPN log
  - firewall log

DB:
  - audit (DDL / DML / login)
```

---

## 4. 도구

| | type | 강점 |
| --- | --- | --- |
| **Splunk** | 상용 | 표준, 강력, 비싸 |
| **Elastic Security** | OSS+상용 | ELK stack 위, free tier |
| **Wazuh** | OSS | host agent + SIEM 무료 |
| **Datadog Cloud Security** | SaaS | observability 통합 |
| **Microsoft Sentinel** | SaaS | Azure 통합 |
| **Chronicle (Google)** | SaaS | enterprise |
| **AWS Security Hub** | SaaS | AWS native, 한정 |
| **CrowdStrike Falcon LogScale** | SaaS | (Humio) |

---

## 5. correlation rule 예

```
1. 짧은 시간 다수 로그인 실패 → brute force
2. 신규 IP / 국가 → 의심
3. root login + IP 변경 → 침해 가능
4. SQL injection 시도 (WAF) + DB 평소와 다른 query
5. EC2 신규 생성 + 비싼 instance type
6. IAM policy 변경 + 새 access key 생성
7. S3 bucket public 으로 변경
8. CloudTrail disable 시도 (★ 침해의 첫 신호)
```

```yaml
# Elastic Security rule
type: query
query: 'event.action: "ssh_login" AND event.outcome: "failure"'
threshold:
  field: source.ip
  value: 10
timeframe: 5m
action: alert
```

---

## 6. MITRE ATT&CK framework

```
공격자의 단계 표준화:
  Initial Access → Execution → Persistence → Privilege Escalation
  → Defense Evasion → Credential Access → Discovery → Lateral Movement
  → Collection → Exfiltration → Impact

각 단계에 detection 매핑.
```

→ SOC 의 표준 reference.

---

## 7. SOAR (Security Orchestration, Automation, Response)

```
SIEM 의 alert → SOAR 가 자동 대응:
  - 1차 분석 (IP reputation 조회)
  - 자동 차단 (firewall rule 추가)
  - ticket 자동 생성
  - playbook 실행

도구:
  - Splunk SOAR (Phantom)
  - Palo Alto Cortex XSOAR
  - Tines
  - Shuffle (OSS)
```

---

## 8. CloudTrail → SIEM 흐름

```
[CloudTrail] → [S3 + SNS] → [Lambda] → [Splunk / Elastic]
       ↓
   [GuardDuty] (이상 탐지 자동)
       ↓
   [Security Hub] (집계)
```

```sql
-- Athena 로 CloudTrail 분석
SELECT eventTime, userIdentity.userName, sourceIPAddress, eventName
FROM cloudtrail_logs
WHERE eventName = 'ConsoleLogin'
  AND responseElements.ConsoleLogin = 'Failure'
  AND eventTime > date_add('hour', -1, current_timestamp)
ORDER BY eventTime DESC;
```

---

## 9. k8s audit log

```yaml
# audit policy
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    resources:
      - {group: "", resources: [secrets, configmaps]}
  - level: RequestResponse
    resources:
      - {group: rbac.authorization.k8s.io, resources: [roles, rolebindings]}
  - level: Metadata
    omitStages: [RequestReceived]
```

→ secret 접근, RBAC 변경 등 log.

---

## 10. retention 정책

```
일반 monitoring log:    7-30일 (cost)
audit log:              1년+ (regulator)
security log:           1년+ (forensic)
compliance:             7년 (SOX, PCI 일부)
```

→ S3 Glacier 등 cheap storage 활용.

---

## 11. PII / GDPR 고려

```
SIEM 이 log 에 PII (email, name) 저장 →
  - encryption at rest
  - access 제한 (SOC 만)
  - retention 정책
  - 사용자 요청 시 삭제

또는 log 에 처음부터 PII 안 넣음:
  - hash (SHA256)
  - 사용자 ID 만 (이름 X)
```

---

## 12. 함정

1. **모든 log 다 — noise** — 정책 + sampling.
2. **rule 너무 적음** — 침해 탐지 X.
3. **rule 너무 많음** — alert fatigue.
4. **PII 그대로 저장** — GDPR 위반.
5. **retention 너무 짧음** — forensic 못 함.
6. **SIEM 자체 노출** — admin credential 도난 시 침묵.
7. **SOC 없음** — alert 만 쌓이고 응답 없음.

---

## 13. 관련

- [[security-ops|↑ security-ops]]
- [[security-incident]]
- [[../monitoring/elk-stack|↗ ELK]]
