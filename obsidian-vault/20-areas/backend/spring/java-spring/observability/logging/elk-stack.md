---
title: "logging/elk-stack — Elasticsearch + Logstash + Kibana"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T07:00:00+09:00
tags: [backend, java-spring, observability, logging, elk, elasticsearch, logstash, kibana]
home_hub: logging
related:
  - "[[logging]]"
  - "[[logback-configuration]]"
  - "[[mdc-logging-filter]]"
  - "[[../monitoring/prometheus-grafana]]"
  - "[[../../../../../devops/monitoring/monitoring]]"
---

# logging/elk-stack — Elasticsearch + Logstash + Kibana

**[[logging|↑ logging]]** · **[[../observability|↑ observability]]**

> 강의 매핑: §5 (ELK 설치 / Logstash / Elasticsearch / 아키텍처) + §6 (Kibana — Discover / Dashboard).

---

## 1. 목적

본 문서는 Spring 응용의 로그를 ELK Stack 으로 중앙 수집 / 검색 / 대시보드 구성하는 처음부터 끝까지의 가이드라인이다.

본 문서가 정의하는 것:
- ELK 의 3 컴포넌트 (Elasticsearch / Logstash / Kibana) 역할
- 아키텍처 — Spring → Logstash → Elasticsearch → Kibana
- Docker Compose 로 ELK 8.x 설치
- Logback → Logstash TCP / TLS 전송
- Logstash pipeline (input / filter / output)
- Elasticsearch index 설계 (date-based / template / lifecycle)
- Kibana Data View / Discover / Dashboard
- 운영 결정 (보안 / 비용 / scaling)

본 문서가 정의하지 않는 것:
- Elasticsearch 자체 운영 깊이 (shard / replica / cluster) — [[../../../../../devops/monitoring/monitoring]]
- Logback 설정 자체 — [[logback-configuration]]
- MDC / request_id — [[mdc-logging-filter]]
- 메트릭 (Prometheus / Grafana) — [[../monitoring/prometheus-grafana]]

---

## 2. ELK 의 3 컴포넌트

| 컴포넌트 | 역할 |
| --- | --- |
| **Logstash** | 로그 수집 / 변환 / 라우팅 (input → filter → output) |
| **Elasticsearch** | 인덱싱 + 검색 + 집계 (시계열 + 전문 검색) |
| **Kibana** | 시각화 — Discover (검색) / Dashboard / Lens / Alert |

→ 4 번째 친구: **Beats** (Filebeat / Metricbeat) — 경량 agent. Logstash 의 부담 줄임.

---

## 3. 아키텍처 — 3 가지 선택지

### 3.1 패턴 A — Logback → Logstash TCP → ES → Kibana (강의 / 권장)

```
[Spring app]
  Logback + LogstashTcpAppender
       │ TCP 5000 (JSON)
       ▼
[Logstash]
  input.tcp → filter.json/grok → output.elasticsearch
       │
       ▼
[Elasticsearch]
  index: spring-2026.05.19
       │
       ▼
[Kibana]
  Data View / Discover / Dashboard
```

| 장점 | 단점 |
| --- | --- |
| application → 즉시 ES (지연 작음) | Logstash 단절 시 application 측 buffer 압박 |
| 단순 (Filebeat 없음) | application 코드의 부담 약간 |
| 권장 — 강의 시나리오 | |

### 3.2 패턴 B — Logback → File → Filebeat → Logstash → ES → Kibana

```
[Spring app]
  Logback FileAppender (JSON)
       │ disk
       ▼
[Filebeat]
  로컬 파일 tail
       │
       ▼
[Logstash] → [ES] → [Kibana]
```

| 장점 | 단점 |
| --- | --- |
| application 무관 (Filebeat 가 file 만 봄) | 단계 많음 / 지연 |
| 디스크 buffer (Logstash 단절 OK) | Filebeat 배포 부담 |

→ container / k8s 환경에서 자주 (stdout → Fluentd / Fluent Bit).

### 3.3 패턴 C — Logback → Kafka → Logstash → ES → Kibana

```
[Spring app] → [Kafka topic logs] → [Logstash kafka input] → [ES] → [Kibana]
```

| 장점 | 단점 |
| --- | --- |
| Kafka 가 buffer / 분리 | Kafka 운영 부담 |
| 대규모 / 다중 소비자 | over-engineering for 작은 시스템 |

→ 본 영역은 **패턴 A** 중심 (강의 기준). 패턴 B / C 는 단순 언급만.

---

## 4. Docker Compose 로 ELK 설치

### 4.1 docker-compose.yml

```yaml
# docker-compose.elk.yml
version: "3.8"
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false           # 학습용 — production 은 활성
      - ES_JAVA_OPTS=-Xms1g -Xmx1g             # 메모리 한도 (host 의 절반 이하)
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9200/_cluster/health || exit 1"]
      interval: 10s
      retries: 12

  logstash:
    image: docker.elastic.co/logstash/logstash:8.13.0
    container_name: logstash
    depends_on:
      elasticsearch:
        condition: service_healthy
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
    ports:
      - "5000:5000/tcp"                         # ★ Logback 의 TCP 입구
      - "9600:9600"                              # monitoring API
    environment:
      LS_JAVA_OPTS: "-Xms512m -Xmx512m"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.0
    container_name: kibana
    depends_on:
      elasticsearch:
        condition: service_healthy
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
      SERVER_PUBLICBASEURL: http://localhost:5601
    ports:
      - "5601:5601"

volumes:
  es-data:
```

### 4.2 디렉토리 구조

```
.
├── docker-compose.elk.yml
└── logstash/
    ├── config/
    │   └── logstash.yml
    └── pipeline/
        └── spring.conf       ← 본 영역의 핵심
```

### 4.3 logstash/config/logstash.yml

```yaml
http.host: 0.0.0.0
xpack.monitoring.enabled: false
log.level: info
```

### 4.4 logstash/pipeline/spring.conf

```ruby
input {
  tcp {
    port => 5000
    codec => json_lines      # LogstashTcpAppender 가 LF-delimited JSON 으로 보냄
  }
}

filter {
  # @timestamp 가 LogstashEncoder 에서 자동 ISO-8601
  # 만약 필요하면 grok / mutate / date 사용

  # 예: stack_trace 분리
  if [stack_trace] {
    mutate {
      add_tag => ["has_error"]
    }
  }

  # 예: 환경 / 서비스 별 라우팅
  mutate {
    add_field => { "[@metadata][index]" => "spring-%{[service]}-%{+YYYY.MM.dd}" }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "%{[@metadata][index]}"
    # production 은 user / password / TLS
  }

  # 디버그용 console 출력 (production 에선 제거)
  # stdout { codec => rubydebug }
}
```

### 4.5 실행

```bash
docker compose -f docker-compose.elk.yml up -d

# 확인
curl -s http://localhost:9200                                    # ES 정상
curl -s http://localhost:9200/_cluster/health | jq .             # status: green/yellow
curl -s http://localhost:5601/api/status                          # Kibana
nc -zv localhost 5000                                              # Logstash TCP

# 로그 확인
docker logs -f logstash
```

---

## 5. Spring 측 — Logback → Logstash TCP

[[logback-configuration]] §7.4 의 LogstashTcpAppender 설정 적용.

```xml
<!-- logback-spring.xml -->
<configuration>
  <appender name="LOGSTASH_TCP" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>${LOGSTASH_HOST:-localhost}:${LOGSTASH_PORT:-5000}</destination>

    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <includeMdcKeyName>request_id</includeMdcKeyName>
      <includeMdcKeyName>user_id</includeMdcKeyName>
      <includeMdcKeyName>tenant_id</includeMdcKeyName>
      <customFields>{"service":"${spring.application.name}","env":"${ENV:-dev}"}</customFields>
    </encoder>

    <reconnectionDelay>1 second</reconnectionDelay>
    <writeBufferSize>16384</writeBufferSize>
    <keepAliveDuration>5 seconds</keepAliveDuration>
  </appender>

  <appender name="ASYNC_LOGSTASH" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="LOGSTASH_TCP"/>
    <queueSize>2048</queueSize>
    <discardingThreshold>0</discardingThreshold>
    <neverBlock>true</neverBlock>             <!-- queue 가득 시 drop — application 정지 안 함 -->
  </appender>

  <springProfile name="prod">
    <root level="INFO">
      <appender-ref ref="CONSOLE"/>
      <appender-ref ref="ASYNC_FILE"/>            <!-- 로컬 file backup -->
      <appender-ref ref="ASYNC_LOGSTASH"/>        <!-- ELK -->
    </root>
  </springProfile>
</configuration>
```

→ 로컬 file + Logstash 양쪽으로 — Logstash 단절 시에도 file backup 보존.

### 5.1 환경 변수

```bash
export LOGSTASH_HOST=logstash.internal
export LOGSTASH_PORT=5000
export ENV=prod
```

---

## 6. Elasticsearch index 설계

### 6.1 date-based index

```
spring-restaurant-api-2026.05.19
spring-restaurant-api-2026.05.20
spring-payment-api-2026.05.19
```

→ 매일 새 index. 보관 정책 / 검색 성능 / 삭제 효율의 표준.

### 6.2 index template — 사전 매핑 정의

```bash
curl -X PUT http://localhost:9200/_index_template/spring-template -H "Content-Type: application/json" -d'
{
  "index_patterns": ["spring-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "index.lifecycle.name": "spring-policy"
    },
    "mappings": {
      "properties": {
        "@timestamp":  { "type": "date" },
        "level":       { "type": "keyword" },
        "logger_name": { "type": "keyword" },
        "message":     { "type": "text" },
        "request_id":  { "type": "keyword" },
        "user_id":     { "type": "keyword" },
        "service":     { "type": "keyword" },
        "env":         { "type": "keyword" },
        "stack_trace": { "type": "text" }
      }
    }
  }
}'
```

| 필드 type | 용도 |
| --- | --- |
| `keyword` | 정확한 검색 / 집계 (level / service / id) |
| `text` | 전문 검색 (message / stack_trace) |
| `date` | 시간 |

### 6.3 ILM (Index Lifecycle Management) — 자동 보관 / 삭제

```bash
curl -X PUT http://localhost:9200/_ilm/policy/spring-policy -H "Content-Type: application/json" -d'
{
  "policy": {
    "phases": {
      "hot":    { "actions": { "rollover": { "max_size": "10gb", "max_age": "1d" } } },
      "warm":   { "min_age": "7d",  "actions": { "shrink": { "number_of_shards": 1 } } },
      "cold":   { "min_age": "30d", "actions": { "freeze": {} } },
      "delete": { "min_age": "90d", "actions": { "delete": {} } }
    }
  }
}'
```

→ 7일 warm / 30일 cold / 90일 삭제. 디스크 사용량 / 검색 성능의 균형.

---

## 7. Kibana — Data View / Discover / Dashboard

### 7.1 Data View 생성

```
Kibana → Stack Management → Data Views → Create
  Name: spring-logs
  Index pattern: spring-*
  Timestamp field: @timestamp
```

### 7.2 Discover — 검색

```
Kibana → Discover → spring-logs 선택

검색 예 (KQL):
  level: "ERROR"
  level: "ERROR" and service: "restaurant-api"
  request_id: "a7c3b421"
  user_id: "user_789" and @timestamp > now-1h
  message: "결제 실패"
  not level: "DEBUG"
```

### 7.3 시간 범위 / refresh

| 옵션 | 사용 |
| --- | --- |
| Quick (last 15m / 1h / 24h / 7d) | 실시간 디버깅 |
| Absolute | 특정 사고 분석 |
| Auto-refresh (5s / 30s) | 실시간 모니터링 |

### 7.4 Dashboard 작성

```
1. Visualize Library → Create visualization → Lens
2. Drag fields onto canvas
3. Save → Add to dashboard

표준 dashboard 4 panel:
- ERROR 빈도 (line chart, count by hour, filter: level=ERROR)
- 서비스 별 로그 수 (pie / bar by service)
- 상위 ERROR 메시지 (data table, top 10 by message)
- 최근 50 ERROR (data table with timestamp / message / request_id)
```

### 7.5 saved query / shared link

| 기능 | 사용 |
| --- | --- |
| Save query | 자주 쓰는 검색 보존 |
| Shareable URL | 사고 시 같은 view 공유 |
| Export CSV | 보고서 |

---

## 8. 운영 결정

### 8.1 보안 (production)

| 항목 | 설정 |
| --- | --- |
| `xpack.security.enabled` | `true` (8.x 기본 true) |
| ES password | `bin/elasticsearch-setup-passwords interactive` |
| Kibana ↔ ES 인증 | `elasticsearch.username` / `elasticsearch.password` |
| Logstash ↔ ES 인증 | output 의 user / password |
| TLS | `--insecure` 금지 — CA + cert |
| Logstash 5000 노출 | private network 만 / mTLS |
| Kibana 외부 노출 | nginx / API Gateway 의 OIDC / SSO |

### 8.2 비용 / scaling

| 규모 | 권장 |
| --- | --- |
| 1 노드 (학습) | single-node ES (위 docker-compose 그대로) |
| 50GB/일 미만 | 3 노드 ES cluster (shard 1, replica 1) |
| 500GB/일 미만 | 5+ 노드 + hot/warm/cold tier + ILM |
| TB/일 | Elastic Cloud / OpenSearch / 자체 cluster 운영 |

→ AWS 의 OpenSearch Service / Elastic Cloud / self-host 비교는 [[../../../../../devops/monitoring/monitoring]].

### 8.3 대안

| 대안 | 특징 |
| --- | --- |
| **Loki** (Grafana) | log aggregation — index by label only, 비용 효율 |
| **Datadog Logs** | SaaS, 완전 managed, 비싸지만 통합 강력 |
| **CloudWatch Logs** | AWS native, 검색 한계 (Logs Insights) |
| **Splunk** | enterprise — 검색 강력, 매우 비쌈 |
| **OpenSearch** | ES 의 오픈 fork (AWS), 호환 |

→ 중소 규모 + 비용 우선 → Loki. 검색 우선 → ELK / OpenSearch.

---

## 9. 트러블슈팅 — 자주 부딪히는 문제

### 9.1 Logstash 가 ES 못 보냄

```bash
docker logs logstash | grep -i error
# Connection refused / timeout

# 1. ES 정상?
curl http://localhost:9200

# 2. Logstash → ES 네트워크
docker exec logstash curl http://elasticsearch:9200
```

### 9.2 Spring app 에서 로그 안 옴

```bash
# 1. TCP 도달?
docker exec -it logstash nc -zv $SPRING_HOST 5000     # 또는 Spring 측에서 logstash:5000

# 2. Logback 의 STATUS 출력 (디버그)
# logback-spring.xml 시작에:
# <statusListener class="ch.qos.logback.core.status.OnConsoleStatusListener"/>

# 3. ES index 생성 여부
curl http://localhost:9200/_cat/indices?v | grep spring
```

### 9.3 디스크 가득 (ES)

```bash
curl http://localhost:9200/_cat/allocation?v
curl http://localhost:9200/_cat/indices?v&s=store.size:desc | head

# 옛 index 강제 삭제
curl -X DELETE http://localhost:9200/spring-2026.04.*
```

→ ILM 정책 필수 (§6.3).

### 9.4 Kibana 가 "no data" 표시

| 원인 | 정정 |
| --- | --- |
| Data View 의 index pattern 잘못 | `spring-*` 검증 |
| 시간 범위가 데이터 밖 | quick → last 24h |
| @timestamp field type 잘못 | mapping 검증 |
| Logstash 가 docs 안 보냄 | §9.1, §9.2 |

---

## 10. 학습 진행 절차

| 단계 | 작업 | 검증 |
| --- | --- | --- |
| 1 | docker-compose up | curl 9200 / 5601 / nc 5000 |
| 2 | logstash pipeline 작성 (§4.4) | docker logs logstash 정상 |
| 3 | Spring 측 LogstashTcpAppender 추가 (§5) | app log 발생 후 ES index 생성 확인 |
| 4 | Kibana Data View 생성 (§7.1) | Discover 에 로그 보임 |
| 5 | 검색 / KQL 연습 (§7.2) | request_id 로 추적 가능 |
| 6 | Dashboard 작성 (§7.4) | 4 panel 표준 |
| 7 | Index template + ILM (§6.2-3) | 자동 회전 / 삭제 |
| 8 | 보안 활성화 (§8.1) | user / TLS |

---

## 11. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| Logstash 단절 시 application 정지 | AsyncAppender 의 neverBlock=false + buffer 가득 | `neverBlock=true` + file backup |
| Elasticsearch 한 node 가 노란색 / 빨간색 | shard 할당 실패 (replica 1 인데 노드 1) | single-node 면 replica=0 |
| index 가 너무 많아 검색 느림 | 매일 + 매 service 마다 → 1000+ index | shard count 조정 / ILM 적극 |
| Logstash filter 가 비활성 (모든 로그 raw) | json 코덱이 LogstashEncoder 와 안 맞음 | `codec => json_lines` 명시 |
| Kibana 에서 새 field 가 안 보임 | Data View refresh 필요 | Stack Management → Data Views → Refresh |
| `text` field 로 정확한 검색 안 됨 | `keyword` 가 아닌 `text` 로 매핑 | template 의 type 검증 |
| 보안 미설정 → 외부 노출 | xpack.security.enabled=false production | 활성화 + auth |
| 한 사고에 ES 가 매우 느림 | shard 너무 큼 (50GB+) | rollover 정책 / max_size |
| timestamp 가 UTC vs 로컬 혼동 | timezone 설정 다름 | 모두 UTC 또는 명시적 +09:00 |

---

## 12. 참고

- [[logging|↑ logging]]
- [[logback-configuration]] §7 — LogstashEncoder
- [[mdc-logging-filter]] — request_id
- [[../monitoring/prometheus-grafana]] — 메트릭 측
- [[../../../../../devops/monitoring/monitoring]] — DevOps 측 ELK 운영
- Elasticsearch docs — https://www.elastic.co/guide/en/elasticsearch/reference/
- Logstash docs — https://www.elastic.co/guide/en/logstash/
- Kibana docs — https://www.elastic.co/guide/en/kibana/
- logstash-logback-encoder — https://github.com/logfellow/logstash-logback-encoder
- 강의: 8 sections 의 §5-6 (ELK + Kibana)
