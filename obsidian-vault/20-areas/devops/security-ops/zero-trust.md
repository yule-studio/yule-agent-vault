---
title: "Zero Trust — BeyondCorp / mTLS / SPIFFE"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:06:00+09:00
tags: [devops, security-ops, zero-trust]
---

# Zero Trust — BeyondCorp / mTLS / SPIFFE

**[[security-ops|↑ security-ops]]**

---

## 1. 개념

> "Never trust, always verify."

전통: 회사 network 안 = 신뢰 (perimeter security, "성벽").  
Zero Trust: 어디에 있든 항상 검증 (VPN 안이라도).

→ Google 의 **BeyondCorp** 가 표준 모델.

---

## 2. 왜 필요

### 왜 필요
- VPN 침투 시 내부 모든 자원 노출.
- 원격 근무 / BYOD / 클라우드 — 명확한 perimeter X.
- supply chain 공격 (3rd party) → 내부.

### 안 하면 문제
- 한 endpoint 침투 → lateral movement → 전체.
- 외부 직원 / 협력사 접근 모호.

### 대안
- VPN — 한 번 통과하면 모두 허용.
- 망 분리 — 운영 부담, hybrid X.

### 트레이드오프
- 운영 복잡도 ↑.
- 사용자 경험 (인증 빈도).

---

## 3. 핵심 요소

```
1. identity 검증 — 누구 (SSO + MFA)
2. device 검증 — 안전한 기기 (MDM)
3. context — 어디서, 언제, 평소 패턴?
4. least privilege — 최소 권한
5. continuous verification — 매 요청
6. encryption — 모든 traffic mTLS
7. monitoring + audit — 이상 탐지
```

---

## 4. BeyondCorp 모델

```
[user laptop]
    ↓ Identity (Google account + MFA)
    ↓ Device cert (MDM 관리)
[Access Proxy (web 입구)]
    ↓ trust score 계산
[application] (회사 network 없음 — 어디에 있든 OK)
```

→ VPN 불필요. 모든 app 이 internet 에 노출되어 있지만 Access Proxy 가 정책 enforcement.

---

## 5. 도구

| | 무엇 |
| --- | --- |
| **Cloudflare Access** | BeyondCorp 구현 (SaaS) |
| **Google IAP** (Identity-Aware Proxy) | GCP |
| **AWS Verified Access** | AWS |
| **Tailscale / Twingate** | mesh VPN + zero trust |
| **Okta + AWS IAM Identity Center** | identity |
| **Cisco Duo** | MFA + device trust |
| **HashiCorp Boundary** | session brokering |

---

## 6. mTLS (service ↔ service zero trust)

```
[service A] ←─ mTLS ─→ [service B]
     ↑                      ↑
   client cert            server cert
   둘 다 검증 → 양쪽 identity 보장
```

```yaml
# Istio (자동)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata: {name: default, namespace: istio-system}
spec:
  mtls: {mode: STRICT}
```

→ Service Mesh 가 mTLS 자동.

---

## 7. SPIFFE / SPIRE (★ workload identity)

```
SPIFFE = 모든 workload 에 고유 ID (SVID)
SPIRE  = SPIFFE 의 구현

ID 예: spiffe://example.com/ns/prod/sa/web

→ k8s SA, Vault, mTLS 가 SPIFFE ID 로 통일
→ "어디서 실행되든" workload 가 자기 자신을 증명
```

---

## 8. AuthorizationPolicy (Istio L7)

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata: {name: api-policy, namespace: prod}
spec:
  selector: {matchLabels: {app: api}}
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/web/sa/web-sa"]
      to:
        - operation:
            methods: [GET]
            paths: ["/api/v1/*"]
```

→ "web service account 만 api 의 GET /api/v1/\* 허용".

---

## 9. JIT (Just-In-Time) access

```
직원이 prod 접근 필요 시:
  1. PR / ticket 으로 요청
  2. approval (peer + 자동 검증)
  3. 1h 한정 권한 부여
  4. 만료 자동 회수
  5. 모든 session 녹화 / audit

도구:
  - HashiCorp Boundary
  - AWS Systems Manager Session Manager
  - Teleport
  - PAM (CyberArk)
```

→ "permanent admin" 없음.

---

## 10. continuous verification

```
세션 중에도 계속 검증:
  - device health (안티바이러스 / patch level)
  - location (이상한 country?)
  - behavior (평소와 다른 패턴?)
  - 이상 시 → step-up auth (MFA 재요청) 또는 차단
```

→ Risk-based authentication.

---

## 11. 실전 단계별 적용

```
Stage 1: SSO + MFA + Conditional Access (Okta / Azure AD)
Stage 2: Device trust (MDM)
Stage 3: Application access proxy (Cloudflare Access / IAP)
Stage 4: Service mesh mTLS (Istio / Linkerd)
Stage 5: SPIFFE / SPIRE
Stage 6: JIT access (Teleport / Boundary)
Stage 7: Continuous risk evaluation
```

→ stage 4까지가 일반. 5+는 enterprise / 금융.

---

## 12. 함정

1. **VPN + zero trust 혼동** — VPN 은 한 번만 검증.
2. **인증 빈도 너무 높음** — 사용자 피로 → 우회 시도.
3. **device trust 부재** — 도난 / 감염 device 접근.
4. **3rd party 공급사** — 같은 zero trust 적용 어려움.
5. **legacy app** — modern auth 미지원 → 별도 처리.
6. **observability 부재** — 침해 탐지 못 함.

---

## 13. 관련

- [[security-ops|↑ security-ops]]
- [[iam-best-practices]]
- [[../networking-ops/service-mesh|↗ Service Mesh]]
- [[../networking-ops/network-policy|↗ Network Policy]]
