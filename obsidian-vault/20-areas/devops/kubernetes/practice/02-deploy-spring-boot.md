---
title: "실습 02 — Spring Boot k8s 배포"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:08:00+09:00
tags: [devops, kubernetes, practice, spring-boot]
---

# 실습 02 — Spring Boot k8s 배포

**[[practice|↑ practice]]**

> 1시간 — docker 실습 02 의 image 를 k8s 에 배포.

---

## 1. image kind 에 load

```bash
docker build -t demo:1.0 .
kind load docker-image demo:1.0 --name dev
```

---

## 2. Deployment + Service + Ingress

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: demo}
spec:
  replicas: 2
  selector: {matchLabels: {app: demo}}
  template:
    metadata: {labels: {app: demo}}
    spec:
      containers:
        - name: app
          image: demo:1.0
          imagePullPolicy: IfNotPresent
          ports: [{containerPort: 8080}]
          resources:
            requests: {cpu: 100m, memory: 256Mi}
            limits:   {cpu: 500m, memory: 512Mi}
          readinessProbe:
            httpGet: {path: /actuator/health/readiness, port: 8080}
            initialDelaySeconds: 20
          livenessProbe:
            httpGet: {path: /actuator/health/liveness, port: 8080}
            initialDelaySeconds: 60
            periodSeconds: 10

---
apiVersion: v1
kind: Service
metadata: {name: demo}
spec:
  selector: {app: demo}
  ports: [{port: 80, targetPort: 8080}]

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service: {name: demo, port: {number: 80}}
```

```bash
kubectl apply -f k8s/
kubectl rollout status deploy/demo
kubectl get pods
curl http://localhost/api/actuator/health
```

---

## 3. PostgreSQL — Bitnami chart

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install pg bitnami/postgresql \
    --set auth.username=demo,auth.password=demopw,auth.database=demo \
    --set primary.persistence.size=2Gi
```

→ `pg-postgresql.default.svc.cluster.local:5432`

```yaml
# Deployment 의 env
env:
  - name: DB_URL
    value: jdbc:postgresql://pg-postgresql:5432/demo
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: pg-postgresql
        key: password
```

---

## 4. Rolling update 확인

```bash
docker build -t demo:1.1 .
kind load docker-image demo:1.1 --name dev
kubectl set image deploy/demo app=demo:1.1
kubectl rollout status deploy/demo
kubectl rollout history deploy/demo
kubectl rollout undo deploy/demo   # 롤백 시
```

---

## 5. HPA

```bash
kubectl autoscale deploy/demo --min=2 --max=10 --cpu-percent=70
kubectl get hpa
```

---

## 6. 정리

```bash
kubectl delete -f k8s/
helm uninstall pg
```

---

## 7. 다음

[[03-helm-chart]]
