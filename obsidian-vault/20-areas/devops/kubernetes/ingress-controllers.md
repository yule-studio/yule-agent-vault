---
title: "Ingress controllers — nginx / Traefik / cert-manager"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:52:00+09:00
tags: [devops, kubernetes, ingress, tls]
---

# Ingress controllers — nginx / Traefik / cert-manager

**[[kubernetes|↑ k8s]]**

---

## 1. 비교

| Controller | 특징 |
| --- | --- |
| **ingress-nginx** ★ | 사실상 표준 — 친숙 |
| **Traefik** | 자동 service discovery + 대시보드 |
| **HAProxy ingress** | 성능 강 |
| **Contour** | Envoy 기반 |
| **AWS LB Controller** | EKS 통합 (ALB direct) |
| **GCP GLBC** | GKE 통합 |
| **Istio Gateway** | service mesh 와 통합 |

→ 본 vault: **ingress-nginx** (단일 cloud) 또는 **AWS LB Controller** (EKS).

---

## 2. ingress-nginx 설치 (Helm)

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
    -n ingress-nginx --create-namespace \
    --set controller.service.type=LoadBalancer
```

---

## 3. cert-manager (Let's Encrypt 자동)

```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
    -n cert-manager --create-namespace \
    --set installCRDs=true
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: {name: letsencrypt-prod}
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef: {name: letsencrypt-prod}
    solvers:
      - http01:
          ingress: {class: nginx}
```

---

## 4. Ingress with TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: {name: web, port: {number: 80}}
  tls:
    - hosts: [api.example.com]
      secretName: api-tls    # cert-manager 자동 생성
```

---

## 5. 주요 annotations (nginx)

```yaml
nginx.ingress.kubernetes.io/proxy-body-size: "10m"
nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
nginx.ingress.kubernetes.io/limit-rps: "10"
nginx.ingress.kubernetes.io/ssl-redirect: "true"
nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"
```

---

## 6. 함정

1. **ingressClassName 누락** → 어떤 controller 가 처리할지 모름.
2. **HTTPS redirect 없음** → HTTP 평문.
3. **rate limit 없음** → DDoS.
4. **단일 ingress 의 host 충돌** → 첫 매치만.
5. **cert-manager + DNS challenge** — 공인 DNS 필요 (sub.example.com).

---

## 7. 관련

- [[kubernetes|↑ k8s]]
- [[services-networking]]
- [[../nginx/nginx|↗ nginx (raw config)]]
