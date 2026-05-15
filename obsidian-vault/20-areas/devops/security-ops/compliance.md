---
title: "Compliance — SOC2 / ISO27001 / PCI-DSS / GDPR / HIPAA"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:14:00+09:00
tags: [devops, security-ops, compliance]
---

# Compliance — SOC2 / ISO27001 / PCI-DSS / GDPR / HIPAA

**[[security-ops|↑ security-ops]]**

---

## 1. 무엇

업종 / 지역의 보안 / 개인정보 / 데이터 처리 요건. 외부 audit 또는 self-attestation.

→ 위반 시 fine / 사업 정지 / 평판 손해.

---

## 2. 주요 표준

| | 범위 | who |
| --- | --- | --- |
| **SOC 2** | 서비스 조직 (B2B SaaS) | AICPA (US) |
| **ISO/IEC 27001** | 정보보안 management | ISO (global) |
| **ISO/IEC 27017/27018** | cloud security | ISO |
| **PCI-DSS** | 카드 결제 | PCI Council |
| **GDPR** | EU 개인정보 | EU |
| **HIPAA** | 의료 (US) | US HHS |
| **K-ISMS / K-ISMS-P** | 한국 정보보호 | KISA |
| **개인정보보호법** | 한국 PII | 정부 |
| **FedRAMP** | 미 정부 cloud | US |
| **CCPA** | California 개인정보 | CA |

---

## 3. SOC 2 (★ B2B SaaS 표준)

5 trust criteria:
- **Security** (필수)
- Availability
- Processing Integrity
- Confidentiality
- Privacy

**Type I** — 시점 — "지금 이 control 이 디자인 되어있나?"  
**Type II** — 기간 (6-12개월) — "실제로 동작했나?"

→ Type II 가 표준 요구. 첫 audit 비용 $30-100k.

흔한 control:
- 접근 통제 (SSO + MFA + IAM least priv)
- audit log (CloudTrail, SIEM)
- vulnerability mgmt (월별 scan)
- 변경 관리 (PR review, CI / CD)
- backup + DR
- incident response (postmortem)
- 직원 training
- vendor mgmt (3rd party 평가)
- physical security (data center → cloud provider 위탁)

---

## 4. PCI-DSS (카드 결제)

12 requirement, 매우 엄격:

```
1. firewall
2. default password 변경
3. 카드 데이터 보호 (저장 시 암호화)
4. 전송 시 암호화 (TLS 1.2+)
5. anti-malware
6. 안전한 system 개발
7. 접근 제한 (need-to-know)
8. 고유 ID + 강한 인증
9. 물리적 접근
10. 모든 접근 log
11. 정기 보안 테스트 (pentest 연 1회 / 분기 scan)
12. 보안 정책
```

→ PAN (카드번호) 저장 = 매우 엄격. → 가능하면 **저장 안 함** (tokenization, PG 사용).

```
Stripe / 이니시스 / 토스 = PCI-DSS Level 1 인증
→ 우리 서버는 tokenization 만 → PCI-DSS scope 축소 (SAQ A-EP)
```

---

## 5. GDPR (EU PII)

```
적용: EU 거주자의 데이터 처리 (회사 위치 무관)

원칙:
  - lawful basis (consent / contract / legal / etc.)
  - data minimization
  - purpose limitation
  - storage limitation (필요한 기간만)
  - transparency
  - 사용자 권리 (access / rectification / erasure / portability)
  - data protection by design
  - breach notification (72h)

위반 fine: 매출의 4% 또는 €20M (최대치)
```

DevOps 요구:
- encryption at rest + in transit
- access log
- 사용자 요청 시 삭제 (right to be forgotten)
- backup 의 삭제도 고려
- data residency (EU 데이터는 EU 에)

---

## 6. HIPAA (US 의료)

```
PHI (Protected Health Information) 보호.
적용: 의료 제공자 / 보험 / business associates.

요구:
  - encryption
  - 접근 통제
  - audit log
  - BAA (Business Associate Agreement) — vendor 와
  - breach notification

위반 fine: $100 ~ $50,000 per record
```

AWS / GCP / Azure — HIPAA 적격 service 제공 + BAA 체결.

---

## 7. K-ISMS-P (한국)

```
정보보호 + 개인정보보호 통합.
연 1회 audit, 3년 인증 갱신.

영역:
  1. 정보보호 정책
  2. 위험 관리
  3. 보호 대책 (관리적 / 물리적 / 기술적)
  4. 사고 대응
  5. 개인정보 처리 (수집 → 파기)
```

→ 한국 SaaS / 공공 / 금융 필수.

---

## 8. compliance as code

```
# OPA / Conftest 으로 정책 검증
package terraform.s3

deny[msg] {
    input.resource.aws_s3_bucket[name]
    not input.resource.aws_s3_bucket[name].acl == "private"
    msg := sprintf("S3 bucket %v must be private", [name])
}

deny[msg] {
    input.resource.aws_db_instance[name]
    not input.resource.aws_db_instance[name].storage_encrypted == true
    msg := sprintf("RDS %v must be encrypted", [name])
}
```

```bash
conftest test terraform.plan.json --policy policies/
```

→ Terraform / k8s manifest 가 정책 위반 시 PR fail.

---

## 9. evidence collection (audit)

```
audit 시 요청:
  - SSO 로그인 log (지난 6개월)
  - access 변경 ticket
  - vulnerability scan 결과
  - patch 정책 이행
  - backup 검증 기록
  - DR drill 결과
  - 직원 training 기록
  - vendor 평가
  - incident postmortem

→ 자동 수집 권장. evidence collection 도구:
  - Drata / Vanta / Tugboat (compliance automation SaaS)
  - 자동 evidence pull (AWS / GitHub / GitLab / Okta)
```

→ Drata / Vanta 가 startup 표준.

---

## 10. 흔한 control 매핑

| 영역 | SOC2 | PCI | GDPR |
| --- | --- | --- | --- |
| encryption rest | CC6 | Req 3 | Art 32 |
| encryption transit | CC6 | Req 4 | Art 32 |
| access control | CC6 | Req 7-8 | Art 32 |
| audit log | CC4 | Req 10 | Art 30 |
| backup | A1 | Req 12 | Art 32 |
| vendor mgmt | CC9 | Req 12 | Art 28 |

→ 한 control 이 여러 표준 동시 충족.

---

## 11. 함정

1. **인증 만 받고 운영 안 함** — Type II 에서 fail.
2. **evidence 수동 수집** — 시간 폭주.
3. **vendor 검토 안 함** — supply chain 책임.
4. **breach notification 72h 못 지킴** — fine.
5. **데이터 삭제 (GDPR) — backup 누락** — 30일+ 남음.
6. **encryption 만, key 평문** — KMS / HSM.
7. **PII 어디 있는지 모름** — data discovery 도구.

---

## 12. 관련

- [[security-ops|↑ security-ops]]
- [[secrets-management]]
- [[iam-best-practices]]
- [[siem]]
