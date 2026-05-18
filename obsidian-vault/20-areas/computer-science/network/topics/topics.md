---
title: "네트워크 토픽 — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:00:00+09:00
tags:
  - network
  - topics
---

# 네트워크 토픽 — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 종합 토픽 인덱스 |
| v.1.1.0 | 2026-05-18 | engineering-agent/tech-lead | DevOps 연결 토픽 3 추가 — container-networking / service-discovery / tcp-socket-ops-pitfalls |

**[[../network|↑ network hub]]**

---

## 1. 한 줄 정의

네트워크의 **종합 / 시스템 레벨** 토픽들 — 여러 layer 를 가로지르는 개념 / 사례 / 추세.

---

## 2. 토픽 인덱스

| 노트 | 영역 |
| --- | --- |
| [[url-to-rendering]] | URL 입력 → 렌더링 (네트워크 전반) |
| [[c10k-c10m]] | C10K / C10M — 동시 연결 한계 |
| [[zero-trust]] | Zero Trust — VPN 의 대체 |
| [[bgp-incidents]] | BGP 사고 사례 (Facebook 2021, AS7007, ...) |
| [[bufferbloat]] | Bufferbloat / BBR / AQM |
| [[dns-load-balancing-anycast]] | DNS-level LB + Anycast |
| [[consistent-hashing]] | 분산 시스템의 consistent hash |
| [[container-networking]] | network namespace / veth / bridge / overlay (VXLAN) — 컨테이너 네트워킹의 OS 기초 |
| [[service-discovery]] | DNS / sidecar / config 기반 service discovery 패턴 |
| [[tcp-socket-ops-pitfalls]] | TIME_WAIT / SO_REUSEPORT / ephemeral port / SYN backlog / conntrack |

---

## 3. 면접 / 토론 단골 토픽

### 시스템 디자인
- "google.com 입력 후 무슨 일?" → [[url-to-rendering]]
- C10K → "수만 동시 connection" → [[c10k-c10m]]
- "백엔드 분산 — 한 backend pin?" → [[consistent-hashing]]

### 운영 사례
- "DNS / BGP 사고 — 회복?" → [[bgp-incidents]]
- "VPN 의 limit?" → [[zero-trust]]
- "고객 latency 변동 큼" → [[bufferbloat]]

---

## 4. 관련

- [[../network]] — 부모 hub
- [[../dns/dns]] / [[../tcp/tcp]] / [[../http/http]] / [[../tls-ssl/tls-ssl]]
