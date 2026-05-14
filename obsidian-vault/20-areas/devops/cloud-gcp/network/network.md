---
title: "GCP Network (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:00:00+09:00
tags:
  - gcp
  - network
  - hub
---

# GCP Network (Hub)

**[[../cloud-gcp|↑ GCP]]**

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **VPC** | [[vpc]] | Global VPC (AWS 와 다른 점) |
| **Cloud Load Balancing** | [[cloud-lb]] | Global LB (L7 / L4) |
| **Cloud CDN** | [[cloud-cdn]] | CDN |
| **Cloud DNS** | [[cloud-dns]] | DNS |

---

## GCP VPC 특이점

- VPC = **글로벌** 자원 (subnet 만 region)
- 같은 VPC 안에서 region 간 private 통신
- AWS VPC 는 region 단위 — 다름

---

## 관련

- [[../cloud-gcp|↑ GCP]]
- [[../../../computer-science/network/network|↗ network 이론]]
