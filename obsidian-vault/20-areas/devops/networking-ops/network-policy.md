---
title: "Network Policy — k8s NetworkPolicy / Cilium"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:56:00+09:00
tags: [devops, networking-ops, network-policy, k8s]
---

# Network Policy — k8s NetworkPolicy / Cilium

**[[networking-ops|↑ networking-ops]]**

---

## 1. 왜 필요

### 왜 필요
- k8s default = **all Pod can talk to all Pod** (flat network).
- 한 service 침투 시 cluster 전체 영향 → zero-trust 필요.

### 안 하면 문제
- web pod 침투 → 직접 DB pod 접속.
- compromise blast radius 전체 cluster.
- 컴플라이언스 (PCI / HIPAA) 위반.

### 대안
- 모든 service 가 mTLS — 무거움.
- 외부 방화벽 — 클러스터 내부 통신 못 막음.

### 트레이드오프
- 운영 부담 (policy 관리).
- 잘못 짜면 service 끊김.

---

## 2. NetworkPolicy 기본

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-from-app
  namespace: prod
spec:
  podSelector:
    matchLabels: {app: postgres}
  policyTypes: [Ingress, Egress]

  ingress:
    - from:
        - podSelector: {matchLabels: {app: web}}
      ports:
        - {protocol: TCP, port: 5432}

  egress:
    - to:
        - namespaceSelector:
            matchLabels: {kubernetes.io/metadata.name: kube-system}
          podSelector:
            matchLabels: {k8s-app: kube-dns}
      ports:
        - {protocol: UDP, port: 53}
```

→ `postgres` pod 는 `web` pod 만 inbound 허용, DNS 만 outbound.

---

## 3. default deny all

```yaml
# namespace 의 모든 pod 가 default deny
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: default-deny-all, namespace: prod}
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

→ 그 다음 명시적 allow rule 만 traffic 통과.

→ **권장 패턴**: deny all → 필요한 것만 allow.

---

## 4. CNI 가 enforce

NetworkPolicy 자체는 spec 만. 실제 enforce 는 CNI plugin:
- **Calico** — 표준 + 강력 (BGP, Global Policy)
- **Cilium** — eBPF, 매우 강력 (L7 정책)
- **Weave Net** — 간단
- **Canal** = Calico + Flannel
- **AWS VPC CNI** — security group for pod (Calico 와 같이)

→ 일부 CNI (flannel only) 는 NetworkPolicy 미지원.

---

## 5. Cilium (eBPF, L7 가능 ★)

```yaml
# CiliumNetworkPolicy — L7 까지
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: {name: api-l7}
spec:
  endpointSelector:
    matchLabels: {app: api}
  ingress:
    - fromEndpoints:
        - matchLabels: {app: web}
      toPorts:
        - ports: [{port: "8080", protocol: TCP}]
          rules:
            http:
              - {method: "GET", path: "/api/v1/.*"}
              - {method: "POST", path: "/api/v1/login"}
```

→ "web 은 api 의 GET / login POST 만 가능". L7 까지 정책.

---

## 6. egress (외부 호출 제한)

```yaml
# Stripe API 만 outbound 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: payment-egress}
spec:
  podSelector: {matchLabels: {app: payment}}
  policyTypes: [Egress]
  egress:
    - to:
        - ipBlock: {cidr: 0.0.0.0/0, except: [10.0.0.0/8, 192.168.0.0/16]}
      ports:
        - {protocol: TCP, port: 443}
    - to:
        - podSelector: {matchLabels: {app: postgres}}
      ports:
        - {protocol: TCP, port: 5432}
```

---

## 7. namespace 간 traffic

```yaml
# prod namespace 의 api 가 monitoring 의 prometheus 와 통신
ingress:
  - from:
      - namespaceSelector:
          matchLabels: {name: monitoring}
        podSelector:
          matchLabels: {app: prometheus}
```

---

## 8. 테스트

```bash
# A pod 에서 B pod 호출
kubectl exec -n prod web-pod -- curl postgres:5432
# → policy OK 면 connection, 아니면 timeout

# Calico
kubectl exec -n prod web-pod -- nc -zv postgres 5432

# Cilium 시각화
hubble observe --from-pod prod/web --to-pod prod/postgres
```

---

## 9. global policy (Calico)

```yaml
# 전체 cluster 에 적용
apiVersion: crd.projectcalico.org/v1
kind: GlobalNetworkPolicy
metadata: {name: deny-egress-internet}
spec:
  selector: tier == "internal"
  egress:
    - action: Deny
      destination:
        notNets: [10.0.0.0/8]
```

---

## 10. 함정

1. **default allow** — namespace 가 deny-all 없으면 모든 pod 가 모든 pod 와 통신.
2. **CNI 가 NetworkPolicy 미지원** (flannel only) — policy 무시.
3. **DNS egress 깜빡** — UDP 53 빠뜨려 service 가 DNS 못 함.
4. **kube-system 차단** — kubelet / metric 영향.
5. **L4 만 — host header 위조** — 같은 port 다른 path 구분 안 됨. Cilium L7 사용.
6. **policy 마이그레이션 = downtime** — staging 검증 후.
7. **observability 없음** — 어떤 traffic 이 deny 됐는지 모름. Hubble (Cilium) / Calico Felix.

---

## 11. 관련

- [[networking-ops|↑ networking-ops]]
- [[service-mesh]]
- [[../kubernetes/kubernetes|↗ k8s]]
- [[../security-ops/security-ops|↗ security-ops]]
