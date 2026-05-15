---
title: "Threat hunting — proactive search / EDR / SIEM rules"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:57:00+09:00
tags: [devops, security-ops, threat-hunting, edr]
---

# Threat hunting — proactive search / EDR / SIEM rules

**[[security-ops|↑ security-ops]]**

---

## 1. 무엇

```
일반 보안:
  alert 발생 → 대응 (reactive)

threat hunting:
  hypothesis-driven proactive search:
  "공격자가 우리 시스템에 이미 있을 것이다.
   어떤 흔적을 남겼을까?"

→ alert 가 없는 침해 찾아냄.
```

---

## 2. 왜 필요

```
평균 dwell time (침해 후 발견):
  2020: 200+ days
  2024: 16 days (개선) 하지만 여전히 큰 risk
  
APT (Advanced Persistent Threat):
  - 검출 회피
  - 정상 도구 사용 (living-off-the-land)
  - 천천히 활동
  - signature-based detection 회피

→ proactive hunting 필요.
```

---

## 3. hunt 의 3 방법

```
1. structured (hypothesis-based)
   "공격자가 X 를 시도한다면 Y 흔적"
   ex: "credential dump → lsass.exe access"

2. unstructured (anomaly)
   normal baseline 벗어남
   ex: "이 host 가 처음 보는 IP 와 통신"

3. situational
   특정 threat intelligence
   ex: "TA505 group 의 IOC 들 우리 환경에?"
```

---

## 4. MITRE ATT&CK framework (★)

```
공격자 행동의 표준화:
  https://attack.mitre.org/

14 tactic (공격 목표):
  1. Reconnaissance
  2. Resource Development
  3. Initial Access
  4. Execution
  5. Persistence
  6. Privilege Escalation
  7. Defense Evasion
  8. Credential Access
  9. Discovery
  10. Lateral Movement
  11. Collection
  12. Command and Control
  13. Exfiltration
  14. Impact

각 tactic 안에 100+ technique.

→ hunt 시 ATT&CK matrix 참고.
```

---

## 5. detection coverage

```
ATT&CK matrix 의 매 cell 마다:
  - 우리 detection 있나?
  - log source 가 충분한가?
  - rule 의 quality?
  - false positive rate?
  
도구:
  - DeTT&CT (Detection coverage)
  - Atomic Red Team (test detection)
  - MITRE Caldera (adversary emulation)
```

---

## 6. log source (★)

```
hunt 의 raw material:

network:
  - VPC Flow Logs
  - DNS query
  - HTTP proxy log
  - WAF log
  - NetFlow

host:
  - Sysmon (Windows)
  - auditd (Linux)
  - osquery
  - eBPF (Falco / Tetragon)
  - EDR agent (CrowdStrike / SentinelOne)

cloud:
  - AWS CloudTrail
  - VPC Flow Logs
  - GuardDuty
  - GCP Cloud Audit Log
  - Azure Activity Log

application:
  - access log
  - auth log
  - admin action
  - error log

→ 다 모이는 곳 = SIEM.
```

---

## 7. EDR (Endpoint Detection and Response)

```
host (server / endpoint) 의 보안 agent:

도구:
  - CrowdStrike Falcon
  - SentinelOne
  - Microsoft Defender for Endpoint
  - Carbon Black
  - SOSafe
  - Wazuh (OSS)
  - osquery + Fleet (OSS)

기능:
  - process / file / network 추적
  - 자동 isolation (의심 시 격리)
  - threat intelligence 통합
  - hunt query (Splunk-like)
```

---

## 8. Falco (★ k8s runtime)

```yaml
# Falco rule
- rule: Shell spawned in container
  desc: shell 실행 detected
  condition: container.id != host and proc.name in (sh, bash, zsh)
  output: "Shell in container (user=%user.name container=%container.name command=%proc.cmdline)"
  priority: WARNING
  tags: [container, shell, mitre_execution]

- rule: Sensitive file access
  condition: open_read and fd.name in (/etc/passwd, /etc/shadow, /root/.ssh/authorized_keys)
  output: "Sensitive file read: %fd.name by %proc.name"
  priority: CRITICAL

- rule: Outbound to known bad IP
  condition: outbound and fd.ip in (threat_intel_ips)
  output: "Outbound to threat IP: %fd.ip"
  priority: CRITICAL
```

→ eBPF / kernel module 으로 syscall 추적.

→ 자동 alert + Tetragon / Cilium 통합.

---

## 9. SIEM hunt query 예

### A. Splunk

```spl
# brute force
index=auth eventtype=login result=failure
| stats count by user, src_ip
| where count > 50
| sort -count

# 신규 admin login (last 7 day)
index=auth eventtype=admin_login
| eval is_new = if(_time > relative_time(now(), "-7d"), 1, 0)
| stats count by user, src_ip, is_new
| where is_new=1

# data exfiltration (큰 outbound)
index=netflow direction=outbound
| stats sum(bytes) as total by src_ip, dest_ip
| where total > 1000000000   # 1GB+
| sort -total
```

### B. Elastic / KQL

```
event.action: "user_login_failure" 
| stats count() by user.name, source.ip
| where count > 50

# DNS anomaly
dns.query.name : ("*.tk" or "*.cf" or "*.ml")
| stats count() by source.ip
```

### C. CloudTrail SQL (Athena)

```sql
-- AccessDenied 폭주
SELECT
    userIdentity.userName as user,
    eventName,
    COUNT(*) as count
FROM cloudtrail
WHERE eventName IS NOT NULL
  AND errorCode = 'AccessDenied'
  AND eventTime > date_add('day', -1, current_timestamp)
GROUP BY 1, 2
ORDER BY count DESC;

-- 새 region 의 API call
SELECT DISTINCT
    userIdentity.userName,
    awsRegion
FROM cloudtrail
WHERE awsRegion NOT IN ('ap-northeast-2', 'us-east-1')
  AND eventTime > date_add('hour', -24, current_timestamp);
```

---

## 10. IOC (Indicator of Compromise)

```
threat intelligence 의 indicator:
  - IP / domain (C2 servers)
  - file hash (SHA256)
  - email
  - URL pattern
  - mutex name (Windows)
  
sources:
  - VirusTotal
  - AlienVault OTX
  - Misp (OSS)
  - Recorded Future
  - Mandiant
  - 정부 (CISA, NCSC)
  - paid threat intel (Crowdstrike / 등)

automation:
  - STIX / TAXII protocol
  - 자동 import → SIEM / EDR
```

---

## 11. hunt 의 시나리오 예

### A. credential stuffing

```
Hypothesis:
  공격자가 leaked credential database 로 시도.

Hunt:
  - 짧은 시간 다수 다른 user 시도
  - 같은 IP / IP range
  - 성공 vs 실패 비율
  - geographic anomaly
  
indicator:
  > 100 login fail in 1 hour from 1 IP
  > 10 different users from same IP in 5 min
```

### B. data exfiltration

```
Hypothesis:
  공격자가 DB / S3 의 data 빼냄.

Hunt:
  - 큰 outbound transfer
  - 새로운 destination IP
  - 새벽 / 비peak 시간
  - 큰 S3 download (CloudTrail)
  - DB query 의 큰 result
```

### C. privilege escalation

```
Hypothesis:
  공격자가 일반 → admin 권한 획득.

Hunt:
  - IAM policy attach (admin)
  - role assume (잘 안 쓰는 role)
  - new access key
  - permission boundary 변경
  - sudoers 변경
```

### D. lateral movement

```
Hypothesis:
  한 host 침해 → 다른 host 침투.

Hunt:
  - SSH from non-bastion
  - admin RDP / SSH between host
  - 비peak SMB share
  - PowerShell remoting (Windows)
  - kubectl exec from compromised pod
```

---

## 12. behavioral analytics (UEBA)

```
User and Entity Behavior Analytics:

baseline 학습 → anomaly 탐지:
  - "alice 가 보통 9-18 시 / 한국 IP"
  - 갑자기 "새벽 3시 / 러시아 IP" → alert
  - "host A 의 평소 outbound 100KB/h"
  - 갑자기 100GB → alert

도구:
  - Exabeam
  - Splunk UBA
  - Microsoft Sentinel UEBA
  - Securonix
  - 자체 ML
```

---

## 13. honeypot / canary

```
가짜 자원으로 침해 즉시 감지:

deployment:
  - canary token (가짜 AWS key, 누가 사용 시 alert)
  - honeyfile (sensitive 파일명, 누가 read 시 alert)
  - honeypot (가짜 RDS / S3)
  - cowrie (가짜 SSH server)

도구:
  - Thinkst Canary
  - 자체 (AWS canary token)

→ 침해 감지 의 가장 빠른 방법.
```

---

## 14. red team / purple team

```
red team:
  공격 시뮬레이션 (외부 또는 내부 team)
  
blue team:
  defense (SOC / DevSecOps)

purple team:
  red + blue 협력
  → red 가 공격 → blue 가 detect 여부
  → 못 잡으면 detection rule 추가
  → 다시 시도
  
→ defense 의 검증.
```

도구:
- MITRE Caldera (adversary emulation)
- Atomic Red Team (small unit test)
- Stratus Red Team (cloud-focused)
- Cobalt Strike (상용, 정식 license)

---

## 15. incident response 와의 관계

```
hunt 가 발견 →
  → incident 발생
  → containment / eradication / recovery
  → postmortem
  → hunt 의 새 rule 추가
  
순환 cycle.
```

---

## 16. ROI

```
hunt 가 가치 있나?

success metric:
  - 발견된 incident (이전에 alert 못 함)
  - mean dwell time 단축
  - detection coverage 확대
  - false positive 감소

team:
  - 작은 회사 = managed (MSSP / MDR)
  - 큰 회사 = 자체 SOC
```

---

## 17. 함정

1. **hunt 만, automation 안 함** — 같은 작업 반복.
2. **log source 부족** — 못 찾음.
3. **noise → 무시** — 진짜 위협 묻힘.
4. **MITRE ATT&CK 무지** — coverage gap 모름.
5. **detection rule 의 stale** — 정기 review.
6. **red team 없음** — defense 검증 안 됨.
7. **PII / privacy** — hunt 의 access 도 audit.
8. **MTTR vs MTTD 균형 X** — detect 만, 응답 부재.

---

## 18. 관련

- [[security-ops|↑ security-ops]]
- [[siem]]
- [[security-incident]]
- [[devsecops]]
