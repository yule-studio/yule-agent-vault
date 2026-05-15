---
title: "Security incident — 침해 사고 대응"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:18:00+09:00
tags: [devops, security-ops, incident]
---

# Security incident — 침해 사고 대응

**[[security-ops|↑ security-ops]]**

---

## 1. 보안 incident 의 특징

일반 [[../sre/incident-response|incident response]] 와 다름:
- 적대적 (의도된 공격자가 있을 수 있음)
- forensics (증거 보존 필수)
- 법적 / 컴플라이언스 영향
- 외부 통보 (사용자 / regulator)
- communication 신중 (공격자가 보고 있을 수도)

---

## 2. NIST IR lifecycle

```
1. Preparation — 도구 / runbook / training / contact list
2. Detection & Analysis — SIEM / alert / 분석
3. Containment — 격리 (단기 + 장기)
4. Eradication — 위협 제거
5. Recovery — 정상 복구
6. Post-incident — postmortem / lesson
```

---

## 3. classification (NIST 800-61)

| | 무엇 | 예 |
| --- | --- | --- |
| **External** | 외부 attacker | brute force, SQLi, DDoS |
| **Internal** | 내부자 | 직원이 데이터 유출 |
| **Accidental** | 실수 | S3 public, secret commit |
| **Malicious code** | malware | ransomware |
| **Improper usage** | 정책 위반 | shared password |
| **Lost / theft** | 분실 | laptop, USB |

---

## 4. severity

| | 영향 | 예 |
| --- | --- | --- |
| **SEV-1 / Critical** | 데이터 유출 확인, prod down, 외부 신고 의무 | 100만+ 사용자 PII 유출 |
| **SEV-2 / High** | 일부 침해 의심, 즉시 차단 | admin 계정 도난, ransomware 시도 |
| **SEV-3 / Medium** | 위협 탐지, 차단 됨 | brute-force 차단, 의심 IP |
| **SEV-4 / Low** | 정보 / 모니터 | scan 시도, single failed login |

---

## 5. containment (격리) 절차

```
즉시 (분 단위):
  - 침해 계정 disable / password reset
  - 침해 host 격리 (network 차단 — VPC SG / 방화벽)
  - access key 회수
  - session token 무효화
  - WAF rule 추가 (공격 IP 차단)

★ 단, 증거 보존:
  - host shutdown X (memory dump 손실)
  - 디스크 image 캡처 (forensic)
  - log 보존 (정상 retention 보다 길게)
```

→ **격리 ≠ shutdown**. 증거 보존 우선.

---

## 6. forensics 도구

```
disk:
  - dd / dcfldd (raw image)
  - FTK Imager
  - Autopsy / Sleuth Kit

memory:
  - LiME (Linux)
  - Volatility (분석)

network:
  - tcpdump / Wireshark
  - VPC Flow Logs
  - NetFlow

cloud:
  - AWS CloudTrail (전체 history)
  - GuardDuty findings
  - AWS Detective (자동 timeline)

container:
  - kubectl logs
  - container snapshot
  - Falco event
```

---

## 7. communication 정책

```
누구에게:
  1. CISO / 보안 책임자 (즉시)
  2. CEO / leadership (Sev-1)
  3. legal (privilege)
  4. PR / 외부 통신 (legal 통과 후만)
  5. customer / regulator (필요 시)

채널:
  - Slack #incident-secure (out-of-band — 침해된 시스템 X)
  - 또는 별도 phone / Signal

원칙:
  - 사실만 (추측 X)
  - 공격자가 볼 수 있다 가정
  - 결정 = leadership
```

---

## 8. breach notification 법적 요구

| 표준 | 시한 |
| --- | --- |
| GDPR | 72h (regulator) |
| HIPAA | 60일 (피해자) |
| CCPA | "가장 빠른 합리적 시간" |
| 한국 개인정보보호법 | 즉시 통지 + 72h 신고 |
| PCI-DSS | 즉시 PG / 카드사 |

→ legal team 과 협업.

---

## 9. ransomware 특별 대응

```
원칙:
  - pay X (FBI 권장)
  - 모든 affected system 격리
  - air-gapped backup 으로 복구
  - 공격 vector 파악 (다시 안 되도록)

★ backup 이 ransomware 의 표적이기도:
  - immutable backup
  - air-gapped (cold storage)
  - 정기 복구 drill
```

---

## 10. data breach 절차

```
1. scope 확인
   - 어떤 데이터? (PII? PCI? PHI?)
   - 얼마나? (몇 명?)
   - 언제부터?

2. 차단 + forensics

3. 법적 자문 (legal first)

4. 통지
   - regulator (GDPR 72h)
   - 피해자
   - 카드사 (PCI)
   - 보험사

5. 조치
   - 영향 user → password reset / token revoke
   - 신용 모니터링 제공 (PII)
   - 보상

6. postmortem (blameless)
   - 어떻게 / 왜
   - control 추가
   - threat model update
```

---

## 11. tabletop exercise

```
가상 incident 시뮬레이션 (분기 1회):
  - "ransomware 가 prod cluster 에"
  - "직원의 GitHub access token 유출"
  - "DB credential commit"

참여: eng / security / legal / comms / leadership

목표:
  - 절차 검증
  - 의사소통 흐름
  - 의사결정자 확인
  - 도구 / 권한 점검
```

---

## 12. 흔한 시나리오 별 runbook

```
- 1. brute force / credential stuffing
- 2. SQL injection 시도
- 3. XSS 발견
- 4. secret commit to git
- 5. S3 bucket public 발견
- 6. ransomware
- 7. employee credential phishing
- 8. supply chain (dependency 침해)
- 9. DDoS
- 10. insider threat
```

각 시나리오마다 detect / contain / eradicate / recover 절차 문서화.

---

## 13. 함정

1. **즉시 shutdown** — 증거 손실 (memory dump 못 함).
2. **공격자 알게 함** — Slack 일반 채널에 논의.
3. **혼자 결정** — legal / leadership 비협의.
4. **breach 인정 늦음** — GDPR 72h 어김.
5. **postmortem 없이 종결** — 같은 침해 반복.
6. **insurance 미가입** — cyber insurance 의 가치.

---

## 14. 관련

- [[security-ops|↑ security-ops]]
- [[siem]]
- [[../sre/incident-response|↗ SRE incident-response]]
- [[../sre/postmortem|↗ postmortem]]
