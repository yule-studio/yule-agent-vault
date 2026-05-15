---
title: "실습 01 — kind 로컬 cluster"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:06:00+09:00
tags: [devops, kubernetes, practice, kind]
---

# 실습 01 — kind 로컬 cluster

**[[practice|↑ practice]]**

> 30분 — Docker 안 k8s cluster 띄우기.

---

## 1. 설치

```bash
# macOS
brew install kind kubectl helm

# Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin
```

---

## 2. Cluster 생성

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - {containerPort: 80, hostPort: 80}
      - {containerPort: 443, hostPort: 443}
  - role: worker
  - role: worker
```

```bash
kind create cluster --name dev --config kind-config.yaml
kubectl cluster-info
kubectl get nodes
# NAME                STATUS   ROLES
# dev-control-plane   Ready    control-plane
# dev-worker          Ready    <none>
# dev-worker2         Ready    <none>
```

---

## 3. ingress-nginx 설치

```bash
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml
kubectl wait --namespace ingress-nginx \
    --for=condition=ready pod \
    --selector=app.kubernetes.io/component=controller \
    --timeout=90s
```

---

## 4. 첫 Deployment

```bash
kubectl create deploy web --image=nginx:1.27-alpine
kubectl expose deploy web --port=80
kubectl get all
```

---

## 5. Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: {name: web}
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: {name: web, port: {number: 80}}
```

```bash
kubectl apply -f ingress.yaml
curl http://localhost
```

---

## 6. 정리

```bash
kind delete cluster --name dev
```

---

## 7. 다음

[[02-deploy-spring-boot]]
