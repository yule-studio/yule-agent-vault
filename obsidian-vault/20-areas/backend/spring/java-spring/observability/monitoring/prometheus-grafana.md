---
title: "monitoring/prometheus-grafana — Prometheus + Grafana + Discord Alert"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T08:00:00+09:00
tags: [backend, java-spring, observability, monitoring, prometheus, grafana, alert]
home_hub: monitoring
related:
  - "[[monitoring]]"
  - "[[actuator-micrometer]]"
  - "[[../logging/elk-stack]]"
  - "[[../../../../../devops/monitoring/monitoring]]"
---

# monitoring/prometheus-grafana — Prometheus + Grafana + Discord Alert

**[[monitoring|↑ monitoring]]** · **[[../observability|↑ observability]]**

> 강의 매핑: §8 (Prometheus 설치 / PromQL / Grafana / Discord Alert).

---

## 1. 목적

본 문서는 Spring Actuator 가 노출한 Prometheus 메트릭을 Prometheus 가 수집하고 Grafana 로 시각화 / 알람하는 처음부터 끝까지의 가이드라인이다.

본 문서가 정의하는 것:
- Prometheus 의 동작 모델 (pull / TSDB / alerting)
- Docker Compose 로 Prometheus + Grafana 설치
- Prometheus 가 Spring Actuator 를 scrape 하는 설정
- PromQL 의 핵심 12 패턴
- Grafana 데이터 소스 + 표준 dashboard
- Grafana Alert + Discord webhook
- production 운영 (보안 / scaling)

본 문서가 정의하지 않는 것:
- Spring Actuator / Micrometer 사용 — [[actuator-micrometer]]
- 로그 (ELK / Loki) — [[../logging/elk-stack]]
- Prometheus 자체 운영 깊이 — [[../../../../../devops/monitoring/monitoring]]

---

## 2. Prometheus 의 동작 모델

### 2.1 pull-based scrape

```
[Prometheus]                           [Spring app 1]
  scrape every 15s ──── HTTP GET ────→ /actuator/prometheus
        ↓
   TSDB (time-series DB)
        ↓
   alert evaluation
        ↓
   Alertmanager → webhook (Discord / Slack / PagerDuty)

   Grafana ─── PromQL query ────→ Prometheus
```

| 특성 | 효과 |
| --- | --- |
| pull-based | Prometheus 가 모든 target 의 health 자체 검증 |
| HTTP GET text format | 어떤 언어 / 시스템도 expose 가능 |
| TSDB (시계열) | 시간 + 값 + label 만 — 검색 빠름 |
| label = key/value | 차원 분석 (group by / filter) |

### 2.2 메트릭 종류

| 종류 | 의미 | 예 |
| --- | --- | --- |
| **counter** | 단조 증가 (재시작 시 reset) | `http_server_requests_seconds_count` |
| **gauge** | 임의 값 | `jvm_memory_used_bytes` |
| **histogram** | 분포 (bucket + count + sum) | `http_server_requests_seconds_bucket` |
| **summary** | 분포 (percentile pre-calculated) | (덜 사용) |

상세는 [[actuator-micrometer]] §4 의 Micrometer instrument.

---

## 3. Docker Compose 로 설치

### 3.1 docker-compose.yml

```yaml
# docker-compose.monitoring.yml
version: "3.8"
services:
  prometheus:
    image: prom/prometheus:v2.51.0
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus-data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=30d            # 30일 보관
      - --web.enable-lifecycle                        # /-/reload endpoint
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:10.4.0
    container_name: grafana
    depends_on: [prometheus]
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
    ports:
      - "3000:3000"

volumes:
  prometheus-data:
  grafana-data:
```

### 3.2 prometheus/prometheus.yml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    env: dev

rule_files:
  - /etc/prometheus/rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: []                # Alertmanager 사용 시 추가 (선택)

scrape_configs:
  # Spring Boot apps
  - job_name: spring-boot
    metrics_path: /actuator/prometheus
    scrape_interval: 15s
    static_configs:
      - targets:
          - host.docker.internal:8081      # Mac/Win — host 의 app 접근
          - host.docker.internal:8082
        labels:
          service: restaurant-api

  # Prometheus 자기 자신
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
```

→ Linux 호스트의 docker compose 면 `host.docker.internal` 대신 `172.17.0.1` 또는 `--add-host=host.docker.internal:host-gateway`.

### 3.3 실행

```bash
docker compose -f docker-compose.monitoring.yml up -d

# 검증
curl http://localhost:9090                                   # Prometheus UI
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job:.labels.job, health:.health}'
# 모든 target 의 health 가 "up" 이면 정상

curl http://localhost:3000                                    # Grafana UI (admin/admin)
```

---

## 4. Spring 측 — 이미 [[actuator-micrometer]] 완료

review:
- `spring-boot-starter-actuator` + `micrometer-registry-prometheus` 의존성
- `management.endpoints.web.exposure.include` 에 `prometheus` 포함
- `management.server.port=8081` 분리
- `/actuator/prometheus` 가 응답하는지 확인

```bash
curl http://localhost:8081/actuator/prometheus | head -30
# # HELP jvm_memory_used_bytes ...
# # TYPE jvm_memory_used_bytes gauge
# jvm_memory_used_bytes{area="heap",id="G1 Eden Space",} 1.342E8
# ...
```

---

## 5. PromQL — 12 핵심 패턴

### 5.1 instant query (현재 값)

```promql
# 현재 모든 Spring app 의 heap 사용량
jvm_memory_used_bytes{area="heap"}

# 특정 서비스
jvm_memory_used_bytes{area="heap", service="restaurant-api"}

# 모두 합
sum(jvm_memory_used_bytes{area="heap"})

# 서비스 별
sum by (service) (jvm_memory_used_bytes{area="heap"})
```

### 5.2 rate — counter 의 초당 증가율

```promql
# 초당 HTTP 요청 수 (RPS) — 5분 윈도우
rate(http_server_requests_seconds_count[5m])

# 서비스 별 RPS
sum by (service) (rate(http_server_requests_seconds_count[5m]))

# URI 별 RPS
sum by (uri) (rate(http_server_requests_seconds_count[5m]))
```

→ `rate` 는 counter 에만 사용. gauge 에는 의미 없음.

### 5.3 increase — 구간 누적 증가

```promql
# 지난 1시간 동안의 5xx 응답 수
increase(http_server_requests_seconds_count{status=~"5.."}[1h])
```

### 5.4 percentile — 응답 시간 p95/p99

```promql
# p95 응답 시간 (5분 윈도우)
histogram_quantile(0.95,
  sum by (le, uri) (
    rate(http_server_requests_seconds_bucket[5m])
  )
)

# p99
histogram_quantile(0.99,
  sum by (le, uri) (
    rate(http_server_requests_seconds_bucket[5m])
  )
)
```

### 5.5 error rate

```promql
# 5xx 비율
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
  /
sum(rate(http_server_requests_seconds_count[5m]))
```

### 5.6 가용 / 한도 비율

```promql
# Hikari pool 사용률
hikaricp_connections_active / hikaricp_connections_max

# Tomcat thread 사용률
tomcat_threads_busy / tomcat_threads_config_max
```

### 5.7 시간 비교

```promql
# 1시간 전 대비 현재 (drift 감지)
(jvm_memory_used_bytes - jvm_memory_used_bytes offset 1h)
  /
jvm_memory_used_bytes offset 1h
```

### 5.8 alert 용 — sustained 비교

```promql
# 5분 동안 5xx 비율 > 5%
(
  sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
    /
  sum(rate(http_server_requests_seconds_count[5m]))
) > 0.05
```

→ Grafana Alert / Prometheus alerting rule 의 표준.

### 5.9 top N

```promql
topk(5, sum by (uri) (rate(http_server_requests_seconds_count[5m])))
```

### 5.10 시간 함수

```promql
# 평균 GC pause (5분)
rate(jvm_gc_pause_seconds_sum[5m]) / rate(jvm_gc_pause_seconds_count[5m])
```

### 5.11 label 변환

```promql
# Counter 이름 / label 변환
label_replace(my_metric, "new_label", "$1", "old_label", "(.+)")
```

### 5.12 4 골든 시그널 (Google SRE)

| 시그널 | PromQL 예 |
| --- | --- |
| **Latency** | `histogram_quantile(0.95, sum by (le) (rate(http_server_requests_seconds_bucket[5m])))` |
| **Traffic** | `sum(rate(http_server_requests_seconds_count[5m]))` |
| **Errors** | `sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))` |
| **Saturation** | `hikaricp_connections_active / hikaricp_connections_max` 또는 `tomcat_threads_busy / max` |

→ 모든 service 의 표준 dashboard.

---

## 6. Grafana — 데이터 소스 + 대시보드

### 6.1 데이터 소스 추가

```
Grafana → Connections → Data Sources → Add → Prometheus
  Name: Prometheus
  URL: http://prometheus:9090
  Save & Test
```

### 6.2 표준 dashboard import — Spring Boot

| dashboard ID (grafana.com) | 설명 |
| --- | --- |
| **6756** | JVM (Micrometer) — JVM 메트릭 통합 |
| **4701** | JVM Dashboard (Micrometer) — popular |
| **14430** | Spring Boot 2.1 System Monitor |
| **17175** | Spring Boot Statistics |

```
Grafana → Dashboards → New → Import → ID 입력
  Data source: Prometheus
```

→ 즉시 JVM / HTTP / Hikari / Tomcat / GC 통합 dashboard.

### 6.3 수동 dashboard 작성 — 4 panel 표준

#### Panel 1 — RPS

```promql
sum by (service) (rate(http_server_requests_seconds_count[1m]))
```

- visualization: Time series
- legend: `{{service}}`
- unit: ops

#### Panel 2 — p95 latency

```promql
histogram_quantile(0.95,
  sum by (le, service) (
    rate(http_server_requests_seconds_bucket[5m])
  )
)
```

- unit: seconds (s)

#### Panel 3 — 5xx error rate

```promql
sum by (service) (rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
  /
sum by (service) (rate(http_server_requests_seconds_count[5m]))
```

- unit: percentunit (0-1)

#### Panel 4 — Hikari pool 사용률

```promql
hikaricp_connections_active / hikaricp_connections_max
```

→ 4 panel = 4 골든 시그널 = service 의 health 즉시 인지.

### 6.4 변수 (variable) 추가

```
Dashboard settings → Variables → Add variable
  Name: service
  Type: Query
  Query: label_values(http_server_requests_seconds_count, service)
  Multi-value: yes
  Include All: yes
```

→ panel 의 `{service="$service"}` 로 사용. 한 dashboard 에서 service 별 필터.

---

## 7. Alert — Grafana → Discord webhook

### 7.1 Discord webhook 발급

```
Discord 서버 → 채널 설정 → 연동 → 웹후크 → 새 웹후크
  이름: Grafana Alert
  채널: #alerts
  URL 복사 — https://discord.com/api/webhooks/.../...
```

### 7.2 Grafana contact point 등록

```
Grafana → Alerting → Contact points → New contact point
  Name: discord-alerts
  Type: Discord
  Webhook URL: <복사한 URL>
  Avatar URL (선택): https://grafana.com/static/img/grafana_logo.png
  Title: '🚨 {{ .CommonLabels.alertname }} - {{ .Status | toUpper }}'
  Message: |
    {{ range .Alerts }}
    **Service**: {{ .Labels.service }}
    **Severity**: {{ .Labels.severity }}
    **Summary**: {{ .Annotations.summary }}
    **Description**: {{ .Annotations.description }}
    **Started**: {{ .StartsAt }}
    **Dashboard**: {{ .Annotations.dashboard }}
    {{ end }}
  Test → Discord 채널에 테스트 메시지 발송 확인
```

### 7.3 Notification policy (라우팅)

```
Grafana → Alerting → Notification policies → Edit default policy
  Default contact point: discord-alerts
  Group by: alertname, service
  Group wait: 30s
  Group interval: 5m
  Repeat interval: 4h
```

### 7.4 Alert rule 작성 — 5xx 비율 > 5%

```
Grafana → Alerting → Alert rules → New alert rule

1. Set query (A)
   Data source: Prometheus
   Query:
     sum by (service) (rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
       /
     sum by (service) (rate(http_server_requests_seconds_count[5m]))

2. Expression (B)
   Reduce: last / mean
   Threshold: > 0.05

3. Set evaluation behavior
   Evaluate every: 1m
   For: 5m            (5분간 지속 시 alert)

4. Configure labels and notifications
   Labels: severity=critical, team=backend
   Annotations:
     summary: "5xx error rate > 5% for {{ $labels.service }}"
     description: "Current rate = {{ $values.B | printf \"%.2f\" }}"
     dashboard: "https://grafana.example.com/d/abc123"

5. Save and exit
```

→ 5분간 5xx 비율이 5% 초과 시 Discord 에 alert.

### 7.5 표준 alert 6 개

| Alert | PromQL | threshold | for |
| --- | --- | --- | --- |
| 5xx error rate | `errors / total` | > 5% | 5m |
| p95 latency | `histogram_quantile(0.95, ...)` | > 1s | 5m |
| heap usage | `jvm_memory_used_bytes{area="heap"} / max` | > 90% | 10m |
| Hikari pool 부족 | `pending > 0` | for | 2m |
| disk usage | `(total - free) / total` | > 85% | 30m |
| service down | `up{job="spring-boot"} == 0` | == 0 | 1m |

---

## 8. Production 운영

### 8.1 보안

| 항목 | 설정 |
| --- | --- |
| Prometheus 외부 노출 | private network 만 / nginx auth |
| Grafana 인증 | OIDC / Google / GitHub / LDAP |
| Spring `/actuator/prometheus` | private port (8081) + IP whitelist |
| Alert webhook URL | 환경변수로 (코드 / git 포함 X) |
| Grafana SECRET_KEY | production 에선 강한 random |

### 8.2 scaling

| 규모 | 권장 |
| --- | --- |
| < 10 service | single Prometheus (현 docker-compose) |
| 10-100 service | Prometheus + Alertmanager 분리 |
| 100+ service | Prometheus federation 또는 Thanos / Cortex / Mimir |
| 멀티 리전 | Thanos / Mimir (글로벌 query) |
| SaaS | Grafana Cloud / Datadog / New Relic |

### 8.3 보관 / 비용

| 옵션 | 효과 |
| --- | --- |
| `--storage.tsdb.retention.time=30d` | 30일 (Prometheus 기본 15일) |
| recording rule | 자주 쓰는 query 미리 계산 → 저장 |
| 외부 storage (Thanos / S3) | 장기 보관 + 비용 효율 |
| Prometheus + Loki + Tempo | Grafana 의 LGTM stack — 통합 비용 ↓ |

### 8.4 통합 dashboard 전략

| dashboard | 용도 |
| --- | --- |
| **Service overview** | RPS / latency / error rate / saturation (4 골든) — service 별 1 row |
| **JVM / GC** | heap / non-heap / GC pause / threads |
| **DB / Cache** | Hikari pool / Redis hit rate / query latency |
| **External API** | client side metrics (latency / error) |
| **Business** | 회원가입 수 / 결제 수 / 알림 발송 등 |
| **SLO / SLI** | error budget / burn rate (별도 — SRE 영역) |

---

## 9. 로그와의 결합

같은 `request_id` 가 로그 (Kibana) 와 trace (Grafana Tempo) 와 메트릭 (Grafana label) 을 연결.

```promql
# Grafana 의 Exemplar — histogram 의 특정 sample 의 request_id 보존
histogram_quantile(0.95, sum by (le) (rate(http_server_requests_seconds_bucket[5m])))
```

→ Exemplar 활성 시 slow request 의 request_id 가 chart 위에 표시. Kibana 의 같은 ID 검색 → application 로그.

상세는 OpenTelemetry / Tempo / Pyroscope 의 통합 — 별도 영역.

---

## 10. 학습 / 적용 절차

| 단계 | 작업 |
| --- | --- |
| 1 | docker-compose 로 Prometheus + Grafana 설치 |
| 2 | prometheus.yml 의 scrape target 에 Spring app 추가 |
| 3 | `/api/v1/targets` 에서 health=up 검증 |
| 4 | Prometheus Web UI 에서 `http_server_requests_seconds_count` 조회 |
| 5 | Grafana 의 Prometheus data source 추가 |
| 6 | dashboard import (ID 4701) — JVM dashboard |
| 7 | 수동 dashboard 작성 — 4 panel (RPS / p95 / error / pool) |
| 8 | Discord webhook 발급 + Grafana contact point 등록 |
| 9 | 5xx > 5% alert rule + test 발사 → Discord 도착 확인 |
| 10 | production — 보안 (별도 port / 인증 / TLS) |

---

## 11. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| target 의 health=down | scrape URL 잘못 / Spring port 잘못 | curl 로 직접 검증 |
| Prometheus 가 `host.docker.internal` 모름 | Linux | `extra_hosts` 또는 host network |
| dashboard 가 "no data" | data source URL / query / 시간 범위 | 각 layer 검증 |
| p95 가 비현실적으로 작음 (전부 5ms) | histogram bucket 부족 | `publishPercentileHistogram()` |
| label cardinality 폭주 | high-cardinality tag (user_id) | tag 검토 — [[actuator-micrometer]] §6.2 |
| Alert 가 한 번 fire 후 안 옴 | repeat interval / silence | notification policy 검증 |
| Discord 메시지 형식 깨짐 | template syntax 잘못 | Grafana template 검증 |
| TSDB 디스크 가득 | retention 길거나 cardinality 큼 | retention 단축 / 메트릭 정리 |
| Grafana admin password 노출 | 환경변수에 plain text | secret manager |
| alert 가 너무 많이 옴 (noise) | threshold 낮음 / for 짧음 | percentile + 5-10분 지속 조건 |

---

## 12. 다음 단계

| 영역 | 가이드 |
| --- | --- |
| Loki / Tempo 통합 (LGTM) | Grafana 의 logs+traces+metrics 단일 view |
| OpenTelemetry | Spring 의 메트릭 + trace + 로그 표준화 |
| SLO / Error Budget | [[../../../../../devops/sre/sre]] |
| 분산 trace | Jaeger / Tempo + OpenTelemetry Java Agent |
| Prometheus Operator (k8s) | PodMonitor / ServiceMonitor / Alertmanager |
| Alertmanager | Prometheus 의 native alert 라우팅 (Grafana 대신 또는 같이) |

---

## 13. 참고

- [[monitoring|↑ monitoring]]
- [[actuator-micrometer]]
- [[../logging/elk-stack]]
- [[../observability]]
- [[../../../../../devops/monitoring/monitoring]]
- [[../../../../../devops/sre/sre]]
- Prometheus 공식 docs — https://prometheus.io/docs/
- PromQL 공식 — https://prometheus.io/docs/prometheus/latest/querying/basics/
- Grafana docs — https://grafana.com/docs/
- Grafana Dashboards 라이브러리 — https://grafana.com/grafana/dashboards/
- Discord Webhook — https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks
- 강의: §8 (Prometheus + Grafana + Discord)
