---
title: "Zero Trust — Never Trust, Always Verify"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:15:00+09:00
tags:
  - network
  - security
  - zero-trust
---

# Zero Trust — Never Trust, Always Verify

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | BeyondCorp / IAP / 모델 |

**[[topics|↑ 토픽 hub]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

**"내부 망 / 위치" 가 신뢰의 기준 X** — 모든 요청 매번 **사용자 / 디바이스 / 컨텍스트 검증**.

---

## 2. 옛 모델 — Castle and Moat

```
외부 (Internet)  ── 방화벽 / VPN ──  내부 (Trusted)
```

### 문제
- 한 번 들어오면 — 자유
- VPN — 인증되면 모든 자원 접근
- 옛 — Pen tester 사례: 한 곳 뚫리면 전부

---

## 3. Zero Trust 의 원칙

| 원칙 | 의미 |
| --- | --- |
| **Never trust** | 위치 / 네트워크 신뢰 X |
| **Always verify** | 매 요청 인증 / 권한 |
| **Least privilege** | 최소 권한 |
| **Assume breach** | 침해 가정 — 격리 / 격벽 |
| **Identity-centric** | 사용자 / 디바이스 중심 |

---

## 4. BeyondCorp — Google (2014+)

### 배경
- 2009 Operation Aurora — Google 망 침해
- VPN 대체 / 직원 in/out 동일 보안

### 구성
- **Device identity** — 매니지드 디바이스
- **User identity** — SSO / MFA
- **Access policy** — 자원 별 정책
- **Access proxy** — 모든 요청 통과

### 흐름
```
직원 → Access Proxy → 검증 (사용자 / 디바이스 / 정책) → 내부 자원
```

→ VPN 없이 — 자유롭게.

---

## 5. 주요 제품

### Cloudflare Access (Zero Trust)
- Cloudflare edge 가 IAM proxy
- SSO 통합 (Google / Okta / Azure AD)
- HTTPS / SSH / RDP / VNC

### Google IAP (Identity-Aware Proxy)
- GCP 자원 — IAM 기반
- Compute Engine / GKE / App Engine

### Tailscale
- WireGuard + Coordination
- ACL — Identity-based

### AWS Verified Access
- IAM Identity Center + 정책

### Zscaler Private Access
- 클라우드 기반 — 엔터프라이즈 큰 규모

### Microsoft Azure AD Application Proxy
- Azure AD 통합

### Banyan / Twingate / Perimeter 81

---

## 6. 핵심 컴포넌트

### IdP — Identity Provider
- Okta / Auth0 / Google / Azure AD
- SAML / OIDC

### MDM — Mobile Device Management
- Jamf / Intune / Kandji
- 매니지드 디바이스 check

### Proxy / Gateway
- HTTPS 양방향
- mTLS / OAuth

### Policy Engine
- 사용자 / 디바이스 / 시간 / 위치 / risk → 허용?
- OPA (Open Policy Agent) / Cedar (AWS)

### Logging / Audit
- 모든 요청 — 누가 / 어디서 / 언제
- SIEM 통합

---

## 7. 정책 예

```yaml
policy:
  - resource: admin.example.com
    require:
      - user.email_domain == "example.com"
      - user.group contains "engineering"
      - device.managed == true
      - device.os.version >= "14"
      - mfa.passed within 8h
      - ip.country == "KR"
```

---

## 8. SDP — Software-Defined Perimeter

### 정의
- Cloud Security Alliance — Zero Trust 의 구현
- "Dark cloud" — 자원이 인증 전 노출 X

### 흐름
1. Client → Controller (인증)
2. Controller → 자원에 client 알림 (SPA — Single Packet Authorization)
3. Client → 자원 (이제 visible)

### 효과
- 자원 — port 닫힘처럼 보임 (DDoS 흡수)
- 인증 후만 connect

---

## 9. mTLS 와의 관계

### 서비스 ↔ 서비스
- 클라이언트도 cert — 양방향 인증
- Service Mesh (Istio / Linkerd) 가 자동

### 사용자 ↔ 자원
- 사용자 — SSO / MFA
- 디바이스 — cert

자세히 → [[../tls-ssl/mtls]]

---

## 10. Zero Trust 의 흐름 — 실제

### SaaS 접근
```
사용자 → Google Workspace
        ↓
IdP (Okta) 인증 + MFA
        ↓
Device check (Intune)
        ↓
Cloudflare Access — 정책 적용
        ↓
Workspace 자원
```

### 사내 도구
```
직원 (집) → tools.example.com
            ↓
Cloudflare Access (인증 / 정책)
            ↓
내부 서비스 (VPN 없이)
```

---

## 11. NIST 800-207 — Zero Trust Architecture

### 7 원칙
1. 모든 자원 — data source / compute
2. 모든 통신 — 안전 (위치 무관)
3. 자원 접근 — per-session
4. 정책 — 사용자 / 디바이스 / 자원 등 다양
5. 자산 무결성 / 보안 모니터링
6. Dynamic 인증 / 권한 (매번)
7. 자산 / 인프라 / 통신 상태 — 데이터 수집

---

## 12. VPN vs Zero Trust

| | VPN | Zero Trust |
| --- | --- | --- |
| **모델** | 네트워크 perimeter | Identity-centric |
| **인증** | 한 번 | 매 요청 |
| **권한** | 전체 망 | 자원 별 |
| **사용자 경험** | 클라이언트 / 느림 | 브라우저 / 빠름 |
| **확장** | 어려움 (server 부담) | 좋음 (edge) |
| **MFA** | 한 번 | 매번 가능 |
| **모니터링** | 어려움 | 강함 |

---

## 13. 도입 단계

### Step 1 — Identity
- SSO / MFA 강제
- Okta / Google / Azure AD

### Step 2 — Inventory
- 자원 / 데이터 분류
- 누가 무엇 필요?

### Step 3 — Policy
- 자원 별 정책

### Step 4 — Proxy / Gateway
- 모든 접근 통과

### Step 5 — Continuous
- 매 요청 검증
- Anomaly detection

---

## 14. 함정

### 함정 1 — IdP 가 SPOF
한 IdP 죽으면 — 모든 자원 접근 X. 다중 IdP / 로컬 fallback.

### 함정 2 — 사용자 경험
매번 인증 — 짜증. SSO + risk-based step-up.

### 함정 3 — Legacy 시스템
옛 시스템 — SAML / OIDC X. 옛 VPN / app 가 필요한 경우 hybrid.

### 함정 4 — 디바이스 변경
새 디바이스 — 등록 / cert 필요. self-service onboarding.

### 함정 5 — Audit log 의 부담
대규모 — 큰 log. 분석 도구 (SIEM).

### 함정 6 — 비용
엔터프라이즈 도구 — 비쌈. SMB — Cloudflare Access (free tier).

### 함정 7 — Cultural shift
"왜 매번 인증?" — 사용자 교육 필수.

---

## 15. 학습 자료

- "BeyondCorp" Google papers
- NIST SP 800-207
- Cloudflare Zero Trust docs
- "Zero Trust Networks" (Gilman, Barth, 2017)
- "Zero Trust Architecture" (Forrester)

---

## 16. 관련

- [[topics]] — Hub
- [[../ssh-rdp-vpn/ssh-rdp-vpn]] — VPN 의 대체
- [[../tls-ssl/mtls]] — 서비스 간
- [[../cdn-proxy/reverse-proxy]] — Access proxy
