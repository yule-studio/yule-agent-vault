---
title: "Loki — log (lightweight)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:36:00+09:00
tags: [devops, monitoring, loki, log]
---

# Loki — log (lightweight)

**[[monitoring|↑ monitoring]]**

---

## 1. 무엇

- "Prometheus for logs" — label index 만, log 본문은 압축.
- Elasticsearch 보다 cost ↓, query 약간 ↓.
- Grafana 와 native 통합.

---

## 2. k8s 설치

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
    -n monitoring \
    --set promtail.enabled=true
```

→ Loki + Promtail (log shipper).

---

## 3. LogQL

```logql
# 모든 web pod 의 log
{app="web"} |= "ERROR"

# JSON parse + filter
{app="web"} | json | level="ERROR" and status >= 500

# rate (Prometheus 처럼)
rate({app="web"} |= "exception" [5m])

# 통계
sum by (status) (count_over_time({app="web"}[5m]))
```

---

## 4. Spring Boot — JSON log

```yaml
# logback-spring.xml
<configuration>
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>
    <root level="INFO">
        <appender-ref ref="JSON"/>
    </root>
</configuration>
```

→ stdout 으로 JSON → k8s 가 file 로 → Promtail 수집 → Loki.

---

## 5. Loki vs ELK

| | Loki | ELK |
| --- | --- | --- |
| 인덱싱 | label 만 | full text |
| 비용 | low | high |
| query 속도 | slow (full scan) | fast (search) |
| 사용 | k8s log / 단순 분석 | 검색 / 분석 / SIEM |

---

## 6. 관련

- [[monitoring|↑ monitoring]]
- [[grafana]]
- [[elk-stack]]
