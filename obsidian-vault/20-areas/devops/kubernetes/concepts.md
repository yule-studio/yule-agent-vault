---
title: "Kubernetes 핵심 개념"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:40:00+09:00
tags: [devops, kubernetes, concepts]
---

# Kubernetes 핵심 개념

**[[kubernetes|↑ k8s]]**

---

## 1. 핵심 리소스

| 리소스 | 역할 | 추상화 |
| --- | --- | --- |
| **Pod** | 1+ 컨테이너 의 단위 | 가장 작은 deployable |
| **Deployment** | Pod 의 desired state (replicas / rolling update) | Stateless |
| **StatefulSet** | Pod 의 stable identity / storage | DB / Redis |
| **DaemonSet** | 모든 node 에 1 Pod | 로그 수집 / monitoring |
| **Job / CronJob** | 1회 / 주기 batch | migration / backup |
| **Service** | Pod 그룹의 stable network endpoint | DNS + LB |
| **Ingress** | HTTP routing (외부 → cluster) | nginx-ingress |
| **ConfigMap / Secret** | 설정 / 비밀 | env / volume |
| **PV / PVC** | 영속 storage | 동적 / 정적 |
| **Namespace** | 논리 격리 | env / team 별 |

---

## 2. Pod = 컨테이너 그룹

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
    - name: app
      image: myapp:1.0
      ports: [{containerPort: 8080}]
    - name: sidecar
      image: log-shipper:1.0
```

- 같은 Pod = 같은 network namespace (`localhost` 로 통신).
- 같은 Pod = 같은 volume.

---

## 3. Deployment (Pod 의 lifecycle 관리)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels: {app: web}
  template:
    metadata:
      labels: {app: web}
    spec:
      containers:
        - name: app
          image: myapp:1.0
          resources:
            requests: {cpu: 100m, memory: 256Mi}
            limits:   {cpu: 500m, memory: 512Mi}
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods -l app=web   # 3 replicas
kubectl rollout status deploy/web
```

---

## 4. Service (Pod → stable endpoint)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector: {app: web}
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP    # ClusterIP | NodePort | LoadBalancer
```

→ `web` DNS 이름 으로 cluster 안 어디서나 접근.

---

## 5. Ingress (외부 → cluster HTTP routing)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port: {number: 80}
  tls:
    - hosts: [api.example.com]
      secretName: tls-cert
```

자세히: [[ingress-controllers]].

---

## 6. ConfigMap vs Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata: {name: app-config}
data:
  application.yml: |
    spring:
      datasource:
        url: jdbc:postgresql://pg:5432/db

---
apiVersion: v1
kind: Secret
metadata: {name: app-secret}
type: Opaque
stringData:
  DB_PASSWORD: supersecret
```

→ Secret 도 base64 encoded — 평문 아님. KMS / sealed-secrets / Vault 권장.

---

## 7. 라벨 / 셀렉터 (모든 거의 기초)

```yaml
metadata:
  labels:
    app: web
    env: prod
    tier: frontend
```

```bash
kubectl get pods -l app=web,env=prod
kubectl delete pods -l env=staging
```

---

## 8. 관련

- [[kubernetes|↑ k8s]]
- [[deployments]]
- [[services-networking]]
- [[configmaps-secrets]]
