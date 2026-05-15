---
title: "실습 02 — Loki + Promtail + Grafana log 통합"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:58:00+09:00
tags: [devops, monitoring, practice, loki]
---

# 실습 02 — Loki + Promtail + Grafana log 통합

**[[practice|↑ practice]]**

---

## 목표

- Loki + Promtail 설치 (k8s)
- Spring Boot 의 stdout JSON log 수집
- Grafana 에서 LogQL 로 query
- log ↔ metric 연결 (Loki + Prom 같은 Grafana)

---

## 1. Loki + Promtail 설치

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
    -n monitoring \
    --set promtail.enabled=true \
    --set grafana.enabled=false       # 이미 kube-prom-stack 의 Grafana 사용
```

Promtail 은 DaemonSet 으로 모든 node 에 배포, `/var/log/pods/*` 수집.

---

## 2. Spring Boot — JSON log 설정

```kotlin
// build.gradle
runtimeOnly("net.logstash.logback:logstash-logback-encoder:7.4")
```

```xml
<!-- src/main/resources/logback-spring.xml -->
<configuration>
    <springProfile name="!dev">
        <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <includeMdcKeyName>trace_id</includeMdcKeyName>
                <includeMdcKeyName>span_id</includeMdcKeyName>
                <includeMdcKeyName>user_id</includeMdcKeyName>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="JSON"/>
        </root>
    </springProfile>
</configuration>
```

→ stdout 으로 한 줄 JSON.

---

## 3. MDC 채우기 (trace_id, user_id)

```kotlin
@Component
class TraceIdFilter : OncePerRequestFilter() {
    override fun doFilterInternal(req: HttpServletRequest, res: HttpServletResponse, chain: FilterChain) {
        val traceId = req.getHeader("X-Trace-Id") ?: UUID.randomUUID().toString()
        MDC.put("trace_id", traceId)
        try {
            chain.doFilter(req, res)
        } finally {
            MDC.clear()
        }
    }
}
```

---

## 4. 배포

```bash
kubectl apply -f deployment.yaml   # 실습 01 의 demo-web 재사용
```

---

## 5. Grafana — Loki data source 추가

```
Configuration → Data sources → Add → Loki
URL: http://loki:3100
Save & test
```

---

## 6. Explore — LogQL

```
Grafana → Explore → Loki 선택
```

```logql
# 1. demo-web 의 모든 log
{namespace="app", pod=~"demo-web-.*"}

# 2. ERROR 만
{namespace="app", pod=~"demo-web-.*"} |= "ERROR"

# 3. JSON parse + level filter
{namespace="app", pod=~"demo-web-.*"} | json | level="ERROR"

# 4. status 별 count_over_time
sum by (level) (count_over_time({namespace="app", pod=~"demo-web-.*"} | json [5m]))

# 5. 특정 trace_id 추적
{namespace="app"} | json | trace_id="abc-def-..."
```

---

## 7. 부하 + 에러 주입

```bash
kubectl run -it --rm load --image=williamyeh/hey -- \
    hey -z 60s -c 5 http://demo-web.app:8080/error
```

→ Grafana Loki Explore 에서 ERROR 로그 stream 확인.

---

## 8. log ↔ metric 연결

같은 Grafana 안에서:
```
Panel 1 (Prometheus): error rate spike
Panel 2 (Loki): {app=demo-web} |= "ERROR" 동시 시각
```

→ click → 같은 시간대 log drilldown.

---

## 9. dashboard JSON (예)

```json
{
  "title": "demo-web — overview",
  "panels": [
    {
      "title": "Error rate",
      "datasource": "Prometheus",
      "targets": [{
        "expr": "sum(rate(http_server_requests_seconds_count{application=\"demo-web\",status=~\"5..\"}[5m]))"
      }]
    },
    {
      "title": "Recent ERROR logs",
      "datasource": "Loki",
      "targets": [{
        "expr": "{namespace=\"app\", pod=~\"demo-web-.*\"} |= \"ERROR\""
      }]
    }
  ]
}
```

---

## 10. 검증 체크리스트

- [ ] Loki Data source 정상
- [ ] LogQL `{namespace="app"}` 결과 stream
- [ ] JSON parse → level / trace_id 추출
- [ ] error 로그 spike 시 Prometheus error rate 와 시간대 일치
- [ ] MDC trace_id 가 모든 log 에 포함

---

## 관련

- [[practice|↑ practice]]
- [[../loki]]
- [[../grafana]]
- [[01-kube-prometheus-stack-setup]]
