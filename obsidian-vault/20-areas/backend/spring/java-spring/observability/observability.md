---
title: "Spring Observability — Logging + Monitoring Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T06:00:00+09:00
tags: [backend, java-spring, observability, logging, monitoring, hub]
home_hub: java-spring
related:
  - "[[../java-spring]]"
  - "[[logging/logging]]"
  - "[[monitoring/monitoring]]"
  - "[[../../../../devops/monitoring/monitoring]]"
---

# Spring Observability — Logging + Monitoring Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-19 | engineering-agent/tech-lead | 최초 — logging (Logback / MDC / ELK) + monitoring (Actuator / Prometheus / Grafana) 통합 hub |

**[[../java-spring|↑ Java + Spring]]**

> Spring Boot 3 기준 — 로그 (Logback → 파일 → ELK) + 모니터링 (Actuator → Prometheus → Grafana → Discord Alert) 의 처음부터 끝까지 완성 흐름.

---

## 1. 영역 정의

본 영역은 Spring Boot 응용의 **관측성 (observability)** 을 처음부터 production 까지 구성하는 가이드라인이다.

본 영역이 정의하는 것:
- 로그 (logging) — 왜 / 무엇을 / Logback 설정 / ELK 수집 / Kibana 모니터링
- 메트릭 (metric) — Spring Actuator / Micrometer / Prometheus / Grafana 대시보드 + Discord Alert
- 두 영역의 결합 — request id (MDC) 가 로그와 trace 를 연결
- Spring 측 코드 + 인프라 측 docker-compose / 설정 파일

본 영역이 정의하지 않는 것:
- 분산 trace (OpenTelemetry / Jaeger / Tempo) — 별도
- ELK / Prometheus 자체의 깊이 운영 — [[../../../../devops/monitoring/monitoring]]
- 알람 정책 (SLO / error budget) — [[../../../../devops/sre/sre]]

---

## 2. 9 lecture roadmap — "처음부터 끝까지"

본 영역은 다음 9 단계 lecture flow 로 구성된다. 각 단계가 별도 노트.

| 단계 | 노트 | 강의 매핑 |
| --- | --- | --- |
| 1 | [[logging/fundamentals]] | §2-3 (로그 필요성 / 무엇을 / Level / 8 구성요소) |
| 2 | [[logging/mdc-logging-filter]] | §3.11 (MDC + UUID + request id 부여 + Servlet Filter) |
| 3 | [[logging/logback-configuration]] | §4 (Logback Appender / 파일 / 압축 / RollingFile) |
| 4 | [[logging/elk-stack]] | §5-6 (Elasticsearch + Logstash + Kibana — Docker / 아키텍처 / 대시보드) |
| 5 | [[monitoring/actuator-micrometer]] | §7 (Actuator endpoint + Micrometer + Prometheus expose) |
| 6 | [[monitoring/prometheus-grafana]] | §8 (Prometheus 설치 + PromQL + Grafana 대시보드 + Discord Alert) |

> 9 lecture = 6 노트 + sub-hub 2 (logging.md, monitoring.md) + 본 hub.

---

## 3. 결합 그래프

```
                                   ┌──────────────────────┐
                                   │   Spring Boot App     │
                                   │                       │
                                   │  Controller / Service │
                                   │       │                │
                                   │       ▼                │
                                   │   Logger (SLF4J)       │
                                   │   + MDC (request_id)   │
                                   │   + Micrometer.counter │
                                   │       │                │
                                   │       ▼                │
                                   │   Logback              │
                                   │       ├─→ console      │
                                   │       ├─→ file (.log.gz)│
                                   │       └─→ Logstash TCP │
                                   │                       │
                                   │   Actuator endpoint   │
                                   │       /actuator/prometheus │
                                   └───────────────────────┘
                                       │           │
                  ┌────────────────────┘           └──────────────────────┐
                  ▼                                                       ▼
        ┌──────────────────┐                                  ┌──────────────────┐
        │  Logstash         │                                  │  Prometheus       │
        │  parse / filter   │                                  │  scrape /metrics  │
        │       │            │                                  │       │            │
        │       ▼            │                                  │       ▼            │
        │  Elasticsearch    │                                  │  TSDB              │
        │  index by date    │                                  │  (시계열)           │
        └──────┬───────────┘                                  └──────┬───────────┘
               ▼                                                      ▼
        ┌──────────────────┐                                  ┌──────────────────┐
        │  Kibana           │                                  │  Grafana          │
        │  Discover /       │                                  │  Dashboard /      │
        │  Dashboard        │                                  │  Alert            │
        └───────────────────┘                                  └──────┬───────────┘
                                                                       ▼
                                                              ┌──────────────────┐
                                                              │  Discord webhook  │
                                                              └──────────────────┘
```

→ 같은 MDC `request_id` 로 application 로그 (Kibana) 와 메트릭 (Grafana) 을 cross-reference.

---

## 4. 의사결정 — 무엇을 어디까지 도입

| 단계 | 도구 | 적용 시점 |
| --- | --- | --- |
| 1. 로그 console + 파일 | Logback (Spring Boot 기본 포함) | 첫 배포부터 |
| 2. 로그 Level / MDC | Logback config + Servlet Filter | 첫 배포부터 |
| 3. 로그 파일 회전 / 압축 | Logback RollingFileAppender | 디스크 한도 가까울 때 |
| 4. ELK 중앙화 | Logstash + Elasticsearch + Kibana | 노드 ≥ 3 또는 디버깅 어려움 |
| 5. 메트릭 노출 | Spring Actuator + Micrometer | 운영 시작 |
| 6. 메트릭 수집 | Prometheus | 메트릭 노출 직후 |
| 7. 대시보드 | Grafana | Prometheus 직후 |
| 8. Alert | Grafana Alert + Discord webhook | 운영 안정성 우선순위 |
| 9. 분산 trace | OpenTelemetry + Jaeger / Tempo | 마이크로서비스 진입 시 |

→ 1-2 는 첫 배포부터. 3-4 는 노드 수 / 트래픽 증가 시. 5-8 은 운영 시작 시 동시.

---

## 5. 환경 / 의존성 요약

### 5.1 build.gradle.kts (Spring 측)

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-actuator")    // 메트릭 endpoint

    implementation("io.micrometer:micrometer-registry-prometheus")              // Prometheus 포맷 export

    implementation("net.logstash.logback:logstash-logback-encoder:8.0")         // Logstash JSON encoder
}
```

### 5.2 docker-compose (인프라 측)

```yaml
# docker-compose.observability.yml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ports: ["9200:9200"]
    volumes: [es-data:/usr/share/elasticsearch/data]

  logstash:
    image: docker.elastic.co/logstash/logstash:8.13.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    ports: ["5044:5044", "5000:5000"]      # 5000 = TCP from Logback
    depends_on: [elasticsearch]

  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports: ["5601:5601"]
    depends_on: [elasticsearch]

  prometheus:
    image: prom/prometheus:v2.51.0
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports: ["9090:9090"]

  grafana:
    image: grafana/grafana:10.4.0
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    ports: ["3000:3000"]
    volumes: [grafana-data:/var/lib/grafana]

volumes:
  es-data:
  grafana-data:
```

→ 각 단계별 자세한 설정은 해당 노트.

---

## 6. 표준 학습 순서

1. [[logging/fundamentals]] — 로그가 무엇이고 왜 필요한가 / Level / 구성요소
2. [[logging/mdc-logging-filter]] — request id 로 로그를 추적 가능하게
3. [[logging/logback-configuration]] — 로그를 파일로 / 회전 / 압축
4. [[logging/elk-stack]] — 다중 서버 로그를 한 곳에 / Kibana 검색·대시보드
5. [[monitoring/actuator-micrometer]] — Spring 측 메트릭 노출
6. [[monitoring/prometheus-grafana]] — 메트릭 수집 + 대시보드 + 알람

각 단계가 이전 단계를 가정. 1-3 은 Spring 측, 4 + 6 은 인프라 측.

---

## 7. 인접 영역과의 책임 분리

| 본 영역 | 인접 영역 | 분리 기준 |
| --- | --- | --- |
| Spring 측 코드 (Logger / Filter / Actuator 사용) | Logback 자체 설정 깊이 / SLF4J 표준 | 본 영역 |
| Logstash / ES / Kibana 의 Spring 통합 | ELK 자체 운영 / 인덱스 튜닝 / 보안 | [[../../../../devops/monitoring/monitoring]] |
| Prometheus / Grafana 의 Spring 통합 | Prometheus 자체 운영 / PromQL 깊이 / Federation | [[../../../../devops/monitoring/monitoring]] |
| Discord webhook | Alert 정책 / SLO / runbook | [[../../../../devops/sre/sre]] |
| 분산 trace | OpenTelemetry / Jaeger / Tempo | (별도 영역, 추가 필요) |

---

## 8. 외부 참고

| 자료 | 영역 |
| --- | --- |
| Spring Boot Actuator 공식 docs | Actuator |
| Micrometer 공식 docs | 메트릭 라이브러리 |
| Logback 공식 docs | logging-classic |
| Elastic Stack docs | ELK |
| Prometheus / Grafana docs | 메트릭 / 대시보드 |
| "Observability Engineering" (Charity Majors et al, O'Reilly) | 일반 |

---

## 9. 관련

- [[../java-spring|↑ Java Spring]]
- [[logging/logging|→ logging sub-hub]]
- [[monitoring/monitoring|→ monitoring sub-hub]]
- [[../../../../devops/monitoring/monitoring|↗ DevOps monitoring (인프라 측)]]
- [[../../../../devops/sre/sre|↗ DevOps SRE]]
