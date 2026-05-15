---
title: "ELK Stack — Elasticsearch + Logstash + Kibana"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:38:00+09:00
tags: [devops, monitoring, elk, log]
---

# ELK Stack — Elasticsearch + Logstash + Kibana

**[[monitoring|↑ monitoring]]**

---

## 1. 구성

```
[App] → Filebeat → Logstash → Elasticsearch → Kibana
                              ↑
                              (또는 직접 OpenSearch — AWS)
```

| | 역할 |
| --- | --- |
| **Elasticsearch** | 검색 / 저장 |
| **Logstash** | 파싱 / 변환 (옵션 — Filebeat 만으로도 가능) |
| **Kibana** | 시각화 / query |
| **Beats** | 경량 shipper (Filebeat / Metricbeat / Heartbeat) |
| **OpenSearch** | AWS fork (라이센스 변경 후) |

---

## 2. KQL (Kibana Query Language)

```
status: 500
service: web AND level: ERROR
@timestamp > "2026-05-14T00:00:00"
message: "stripe* timeout"
```

---

## 3. 인덱스 패턴 / lifecycle

```
logs-web-2026-05-15
logs-web-2026-05-16
```

- ILM (Index Lifecycle Management) — hot → warm → cold → delete.
- 비용 절감.

---

## 4. ELK vs Loki

| | ELK | Loki |
| --- | --- | --- |
| 인덱싱 | 전체 텍스트 | label 만 |
| query 속도 | 빠름 | 느림 (full scan) |
| 비용 | 비쌈 (CPU + storage) | 저렴 |
| 사용 | SIEM / 검색 / 분석 | k8s 일반 log |

---

## 5. ELK vs Datadog / CloudWatch

- ELK = OSS / 직접 운영 (또는 Elastic Cloud).
- Datadog Logs = SaaS.
- AWS CloudWatch Logs = AWS 통합.

---

## 6. 함정

1. **shard 수 잘못** — 너무 많음 (overhead) / 적음 (확장 X).
2. **ILM 없음** — 인덱스 무한 누적.
3. **mapping explosion** — 동적 field 폭증.
4. **Filebeat → ES 직접** — Logstash buffer 없으면 손실.
5. **OpenSearch / Elasticsearch 라이센스** — SaaS 사용 시 확인.

---

## 7. 관련

- [[monitoring|↑ monitoring]]
- [[loki]]
- [[../security-ops/security-ops|↗ SIEM]]
