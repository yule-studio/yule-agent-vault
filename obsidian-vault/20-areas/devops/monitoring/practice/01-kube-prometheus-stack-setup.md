---
title: "실습 01 — kube-prometheus-stack + Spring scrape"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:56:00+09:00
tags: [devops, monitoring, practice, prometheus, grafana]
---

# 실습 01 — kube-prometheus-stack + Spring scrape

**[[practice|↑ practice]]**

---

## 목표

- k8s 에 Prometheus + Grafana + AlertManager 설치
- Spring Boot 앱이 `/actuator/prometheus` 노출
- ServiceMonitor 로 scrape
- Grafana 에서 RED metric 대시보드 확인

---

## 1. cluster 준비

```bash
kind create cluster --name obs

kubectl create namespace monitoring
kubectl create namespace app
```

---

## 2. kube-prometheus-stack 설치

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kps prometheus-community/kube-prometheus-stack \
    -n monitoring \
    --set grafana.adminPassword=admin123 \
    --set grafana.service.type=NodePort
```

→ Prometheus + Grafana + AlertManager + node-exporter + kube-state-metrics 자동.

---

## 3. Grafana 접속

```bash
kubectl port-forward -n monitoring svc/kps-grafana 3000:80
```

브라우저 `http://localhost:3000` (admin / admin123).

내장 dashboard:
- Kubernetes / Compute Resources / Pod
- Node Exporter / Nodes

---

## 4. Spring Boot 샘플 앱

```kotlin
// build.gradle
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")
}
```

```yaml
# application.yml
spring:
  application.name: demo-web

management:
  endpoints.web.exposure.include: prometheus,health
  metrics.tags.application: ${spring.application.name}
```

```kotlin
@RestController
class DemoController {
    @GetMapping("/hello")
    fun hello() = "world"

    @GetMapping("/error")
    fun error(): String {
        throw RuntimeException("boom")
    }
}
```

---

## 5. k8s manifest

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-web
  namespace: app
spec:
  replicas: 2
  selector: {matchLabels: {app: demo-web}}
  template:
    metadata:
      labels: {app: demo-web}
    spec:
      containers:
        - name: demo-web
          image: ghcr.io/yourname/demo-web:1.0.0
          ports:
            - {name: http, containerPort: 8080}
          readinessProbe:
            httpGet: {path: /actuator/health, port: 8080}
---
apiVersion: v1
kind: Service
metadata:
  name: demo-web
  namespace: app
  labels: {app: demo-web}
spec:
  selector: {app: demo-web}
  ports:
    - {name: http, port: 8080, targetPort: 8080}
```

```bash
kubectl apply -f deployment.yaml
```

---

## 6. ServiceMonitor (Prometheus Operator)

```yaml
# service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: demo-web
  namespace: monitoring     # Prometheus 같은 namespace
  labels:
    release: kps            # kube-prometheus-stack 의 selector
spec:
  namespaceSelector:
    matchNames: [app]
  selector:
    matchLabels: {app: demo-web}
  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 15s
```

```bash
kubectl apply -f service-monitor.yaml
```

---

## 7. Prometheus 확인

```bash
kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-prometheus 9090
```

`http://localhost:9090/targets` → demo-web/0 UP.

PromQL:
```promql
http_server_requests_seconds_count{application="demo-web"}
```

---

## 8. 부하 주입

```bash
kubectl run -it --rm load --image=williamyeh/hey -- \
    hey -z 60s -c 10 http://demo-web.app:8080/hello

kubectl run -it --rm load2 --image=williamyeh/hey -- \
    hey -z 60s -c 5 http://demo-web.app:8080/error
```

---

## 9. Grafana 대시보드 — RED

```
Dashboard → Import → ID 11378 (Spring Boot 2.1 Statistics)
```

또는 manual panel:

```promql
# RPS
sum by (uri, status) (rate(http_server_requests_seconds_count{application="demo-web"}[1m]))

# p95 latency
histogram_quantile(0.95,
    sum by (le, uri) (rate(http_server_requests_seconds_bucket{application="demo-web"}[5m]))
)

# Error rate
sum(rate(http_server_requests_seconds_count{application="demo-web",status=~"5.."}[5m]))
/ sum(rate(http_server_requests_seconds_count{application="demo-web"}[5m]))
```

---

## 10. alert rule

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: demo-web-rules
  namespace: monitoring
  labels: {release: kps}
spec:
  groups:
    - name: demo-web
      rules:
        - alert: DemoWebHighErrorRate
          expr: |
            sum(rate(http_server_requests_seconds_count{application="demo-web",status=~"5.."}[5m]))
            / sum(rate(http_server_requests_seconds_count{application="demo-web"}[5m]))
            > 0.01
          for: 2m
          labels: {severity: warning}
          annotations:
            summary: "demo-web error rate > 1%"
```

→ AlertManager UI 에서 확인 가능.

---

## 11. 정리

```bash
kind delete cluster --name obs
```

---

## 검증 체크리스트

- [ ] Grafana 접속 (admin/admin123)
- [ ] Prometheus `/targets` 에 demo-web/0 UP
- [ ] `http_server_requests_seconds_count` query 결과 있음
- [ ] RED panel 3개 작동
- [ ] alert "DemoWebHighErrorRate" firing 확인

---

## 관련

- [[practice|↑ practice]]
- [[../prometheus]]
- [[../grafana]]
- [[../application-metrics]]
