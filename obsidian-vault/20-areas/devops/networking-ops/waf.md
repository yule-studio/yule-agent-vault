---
title: "WAF — Web Application Firewall"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:54:00+09:00
tags: [devops, networking-ops, waf, security]
---

# WAF — Web Application Firewall

**[[networking-ops|↑ networking-ops]]**

---

## 1. 무엇

L7 보안 — HTTP 요청을 검사 + 차단:
- SQL injection
- XSS (Cross-Site Scripting)
- RCE (Remote Code Execution)
- LFI / RFI (File Inclusion)
- bot / scraping
- DDoS (L7)
- OWASP Top 10

→ 일반 방화벽 (L3/L4) 와 다름. **WAF = application layer**.

---

## 2. 도구

| | type | 강점 |
| --- | --- | --- |
| **AWS WAF** | SaaS | ALB / CloudFront / API GW 통합 |
| **Cloudflare WAF** | SaaS | OWASP CRS + ML, 무료 tier |
| **Azure WAF** | SaaS | Application Gateway 통합 |
| **GCP Cloud Armor** | SaaS | GCP LB 통합 |
| **F5 / Imperva** | enterprise | on-prem hardware |
| **ModSecurity** | OSS | nginx / Apache, OWASP CRS |
| **NAXSI** | OSS | nginx, 가벼움 |
| **Coraza** | OSS | ModSecurity 의 Go fork |

---

## 3. OWASP Core Rule Set (CRS)

```
가장 많이 쓰는 rule set (Apache foundation).
ModSecurity / Coraza 의 표준.

분류:
  900: initialization
  901: scan detection
  910: IP reputation
  920: protocol enforcement
  930: file inclusion (LFI/RFI)
  931: RFI
  932: RCE
  933: PHP injection
  941: XSS
  942: SQL injection
  944: Java
  949: blocking
```

→ paranoia level 1-4 (높을수록 엄격, false positive ↑).

---

## 4. AWS WAF rule 예

```yaml
# Managed rule
- AWSManagedRulesCommonRuleSet           # OWASP Core
- AWSManagedRulesKnownBadInputsRuleSet
- AWSManagedRulesSQLiRuleSet
- AWSManagedRulesAmazonIpReputationList

# Custom rule
- name: rate-limit-login
  statement:
    rate-based:
      limit: 100              # 5분 window 100 req
      aggregate-key: IP
      scope-down:
        byte-match:
          uri: { contains: "/login" }
  action: block
```

---

## 5. Cloudflare WAF

```
1. Managed Rules (자동 OWASP)
2. Custom Rules (expression)
   - (http.request.uri.path eq "/admin" and ip.src ne 1.2.3.4)
3. Rate Limiting
4. Bot Fight Mode
5. Country block
```

---

## 6. ModSecurity (nginx) 예

```bash
# 설치 (Ubuntu)
sudo apt install libnginx-mod-http-modsecurity

# CRS 다운로드
git clone https://github.com/coreruleset/coreruleset /etc/nginx/modsec/crs
cp /etc/nginx/modsec/crs/crs-setup.conf.example \
   /etc/nginx/modsec/crs/crs-setup.conf
```

```
# /etc/nginx/modsec/main.conf
SecRuleEngine On
SecRequestBodyAccess On
SecResponseBodyAccess Off
SecAuditEngine RelevantOnly
SecAuditLog /var/log/nginx/modsec_audit.log

Include /etc/nginx/modsec/crs/crs-setup.conf
Include /etc/nginx/modsec/crs/rules/*.conf
```

```nginx
http {
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;
}
```

---

## 7. test (intentional attack)

```bash
# SQL injection 시도
curl "https://example.com/search?q=' OR 1=1--"
# WAF → 403 Forbidden / blocked

# XSS
curl "https://example.com/?q=<script>alert(1)</script>"

# RCE
curl "https://example.com/?cmd=;cat /etc/passwd"
```

→ 응답이 정상이면 WAF 동작 안 함.

---

## 8. mode

| | 무엇 |
| --- | --- |
| **detect / monitor / log only** | 차단 X, log 만 (튜닝 단계) |
| **block** | 실제 차단 |
| **challenge** | CAPTCHA / JS challenge (Cloudflare) |

→ 처음 도입 시 detect 으로 false positive 확인 → block.

---

## 9. false positive 처리

```
1. WAF log 분석 — 어떤 rule 이 trigger
2. 정당한 traffic 인지 확인
3. exception 추가:
   - 특정 IP / path / user 만 우회
   - 특정 rule disable
   - rule 의 anomaly score 조정
4. 적용 후 모니터링
```

```yaml
# AWS WAF exception
- statement:
    and:
      - byte-match:
          uri: { contains: "/api/upload" }
      - not:
          managed-rule-group: SQLiRuleSet
  action: allow
```

---

## 10. WAF + CDN + LB 조합

```
[user] → [Cloudflare WAF + CDN] → [AWS ALB + AWS WAF] → [Spring + ModSecurity]
              ↑                          ↑                      ↑
              DDoS / bot                 SQLi / XSS              app-level (defensive depth)
```

→ **depth in defense** — 한 layer fail 해도 다음 layer.

---

## 11. 함정

1. **false positive 차단** — 정상 traffic 영향. detect → block 점진.
2. **WAF 만 의존** — application 보안 (input validation) 도 필수.
3. **rule update 안 함** — managed rule auto-update 켜기.
4. **OWASP level 4 paranoia 처음부터** — 너무 엄격, 정상 차단.
5. **body size limit** — 큰 upload 시 false positive.
6. **log 검토 안 함** — 공격 시도 모름.
7. **cost** — managed WAF 는 request 당 과금 (큰 traffic 비쌈).

---

## 12. 관련

- [[networking-ops|↑ networking-ops]]
- [[cdn]]
- [[../nginx/security|↗ nginx security]]
- [[../security-ops/security-ops|↗ security-ops]]
