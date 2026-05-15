---
title: "SSL/TLS 운영 — cert 관리 / ACM / cert-manager"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:58:00+09:00
tags: [devops, networking-ops, tls, cert]
---

# SSL/TLS 운영 — cert 관리 / ACM / cert-manager

**[[networking-ops|↑ networking-ops]]**

---

## 1. cert 관리 방법

| | 무엇 | 사용 |
| --- | --- | --- |
| **AWS ACM** | AWS managed, 무료 | ALB / CloudFront / API GW |
| **GCP Certificate Manager** | GCP managed | GCP LB |
| **Azure Key Vault** | Azure | App Gateway |
| **Let's Encrypt + certbot** | OSS, 무료 | nginx server |
| **Let's Encrypt + cert-manager** | k8s | Ingress |
| **상용 CA** (DigiCert, Sectigo) | 유료 | EV cert / wildcard |
| **자체서명** | 개발 / 내부 | (외부 사용 X) |

→ **외부 사용 = Let's Encrypt 또는 ACM 무료**. 유료는 EV / 특수 케이스만.

---

## 2. Let's Encrypt 한계 / 특징

```
- 무료
- 90일 유효 (★ 자동 갱신 필수)
- wildcard 가능 (DNS challenge)
- ACME protocol
- rate limit:
    * 같은 domain 주 50개
    * duplicate cert 주 5개
    * 실패 시 1시간 / IP
```

→ rate limit 걸리면 staging endpoint (`acme-staging-v02.api.letsencrypt.org`) 로 테스트.

---

## 3. AWS ACM

```bash
# 발급 (DNS validation 권장)
aws acm request-certificate \
    --domain-name example.com \
    --subject-alternative-names "*.example.com" \
    --validation-method DNS

# Route53 자동 등록
aws acm list-certificates
```

→ ACM 은 ALB / CloudFront / API GW 에만 사용 가능 (EC2 직접 사용 X).

→ 자동 갱신 (60일 전).

---

## 4. cert-manager (k8s ★)

```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
    -n cert-manager --create-namespace \
    --set installCRDs=true
```

```yaml
# Issuer (namespace) 또는 ClusterIssuer (cluster-wide)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: {name: letsencrypt-prod}
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef: {name: letsencrypt-prod-key}
    solvers:
      - http01:
          ingress: {class: nginx}
      - dns01:
          route53: {region: us-east-1}
        selector:
          dnsZones: [example.com]
```

```yaml
# Certificate 자동 발급 + 갱신
apiVersion: cert-manager.io/v1
kind: Certificate
metadata: {name: example-com, namespace: prod}
spec:
  secretName: example-com-tls
  duration: 2160h         # 90d
  renewBefore: 720h       # 30d 전 갱신
  issuerRef: {name: letsencrypt-prod, kind: ClusterIssuer}
  dnsNames:
    - example.com
    - "*.example.com"
```

```yaml
# Ingress annotation 으로 자동
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts: [example.com]
      secretName: example-com-tls
  rules:
    - host: example.com
      http:
        paths: [{path: /, pathType: Prefix, backend: ...}]
```

→ Ingress 만 만들면 cert 자동 발급 / 갱신.

---

## 5. ACME challenge

| | 무엇 | 장단 |
| --- | --- | --- |
| **HTTP-01** | `/.well-known/acme-challenge/...` 에 파일 | 간단, wildcard X |
| **DNS-01** | TXT record 추가 | wildcard OK, DNS 자동화 필요 |
| **TLS-ALPN-01** | TLS handshake | port 443 만 열려있을 때 |

→ wildcard (`*.example.com`) 는 DNS-01 만 가능.

---

## 6. cert chain 검증

```bash
# fullchain
openssl s_client -connect example.com:443 -showcerts

# expire 확인
echo | openssl s_client -connect example.com:443 2>/dev/null | \
    openssl x509 -noout -dates

# 모든 host
for host in api web admin; do
    expire=$(echo | openssl s_client -connect $host.example.com:443 -servername $host.example.com 2>/dev/null | openssl x509 -noout -enddate)
    echo "$host: $expire"
done
```

---

## 7. 모니터링

```promql
# Prometheus (cert-manager + Prometheus exporter)
certmanager_certificate_expiration_timestamp_seconds - time() < 7 * 24 * 3600
```

→ "7일 안에 만료" alert.

```yaml
# blackbox exporter
- module: tcp_tls
  targets: [example.com:443]
```

---

## 8. mTLS (mutual TLS)

```
client 도 cert 제시 → server 가 검증.

사용:
  - B2B API
  - 내부 service ↔ service (service mesh 자동)
  - 금융 / 정부 API
```

```nginx
ssl_client_certificate /etc/nginx/ca.crt;
ssl_verify_client on;
```

---

## 9. private CA

```
회사 내부용 — 외부 CA 가 아닌 자체 CA.

도구:
  - HashiCorp Vault PKI
  - AWS Private CA
  - smallstep CA (OSS)
  - cfssl (Cloudflare)
```

→ service mesh, mTLS 의 root.

---

## 10. 함정

1. **자동 갱신 hook 누락** — cert 만 갱신, nginx reload 안 함.
2. **wildcard 없이 sub-domain 폭증** — 각 도메인마다 cert.
3. **HTTP-01 challenge path block** — `.well-known/` 못 접근.
4. **rate limit (Let's Encrypt)** — 같은 도메인 주 50회.
5. **intermediate 빠짐** — 일부 client 실패.
6. **cert ↔ key mismatch** — 매번 새 key 발급 후 옛 cert.
7. **cert 만료 모니터링 없음** — 갑자기 서비스 down.
8. **HSTS 잘못 등록** — 잘못된 cert 시 1년 사용자 잠금.

---

## 11. 관련

- [[networking-ops|↑ networking-ops]]
- [[dns]]
- [[../nginx/tls-https|↗ nginx TLS]]
- [[../security-ops/security-ops|↗ security-ops]]
