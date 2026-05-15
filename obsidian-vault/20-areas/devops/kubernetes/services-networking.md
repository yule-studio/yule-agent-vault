---
title: "Services + Networking"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:44:00+09:00
tags: [devops, kubernetes, network, service]
---

# Services + Networking

**[[kubernetes|↑ k8s]]**

---

## 1. Service type

| Type | 사용 |
| --- | --- |
| **ClusterIP** ★ | cluster 안 통신 (default) |
| **NodePort** | 노드 IP:port 노출 (dev only) |
| **LoadBalancer** | cloud LB 자동 생성 (AWS NLB / GCP LB) |
| **ExternalName** | DNS alias (외부 service) |
| **Headless** (clusterIP=None) | DNS round-robin (StatefulSet) |

```yaml
apiVersion: v1
kind: Service
metadata: {name: web}
spec:
  type: ClusterIP
  selector: {app: web}
  ports: [{port: 80, targetPort: 8080}]
```

---

## 2. Service DNS

```
<service>.<namespace>.svc.cluster.local
        ↓
web.production.svc.cluster.local
```

같은 namespace 면 짧게: `web`.

---

## 3. Ingress (HTTP routing)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend: {service: {name: web, port: {number: 80}}}
  tls:
    - hosts: [api.example.com]
      secretName: tls-cert
```

---

## 4. NetworkPolicy (격리)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: deny-all}
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

→ default deny + 명시 allow.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: allow-web-to-db}
spec:
  podSelector: {matchLabels: {app: db}}
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: {matchLabels: {app: web}}
      ports: [{port: 5432}]
```

---

## 5. Service Mesh (옵션, 큰 cluster)

- **Istio** — feature 강 (mTLS / circuit breaker / canary)
- **Linkerd** — 가벼움
- **Cilium** — eBPF 기반
- 본 vault: 시작 시 없음. F12+ 검토.

---

## 6. 함정

1. **default deny X** → 모든 Pod 간 통신 가능.
2. **LoadBalancer 무한 생성** → cloud 비용.
3. **NodePort production** — IP / port exposed.
4. **Service selector 잘못** → 0 endpoints.
5. **DNS caching** — Service IP 변경 후 옛 IP 사용.

---

## 7. 관련

- [[kubernetes|↑ k8s]]
- [[ingress-controllers]]
- [[concepts]]
