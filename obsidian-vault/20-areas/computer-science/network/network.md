---
title: "네트워크 (Network) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T10:00:00+09:00
tags:
  - network
  - hub
  - tcp-ip
  - http
  - osi
---

# 네트워크 (Network) — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.2.0.0 | 2026-05-13 | engineering-agent/tech-lead | hub 재편 + 깊이 분리 시작 (OSI 7 계층 / TCP / HTTP / TLS / DNS / 라우팅 등 각각 별도 폴더) |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 단일 노트 (큰 틀) |

**[[../computer-science|↑ computer-science]]**

> 이 노트는 **네트워크 전체의 hub**. 각 개념은 별도 폴더 / 노트로 깊이 다룬다.
> 단일 사전이었던 v.1.0.0 은 hub 로 재편되어 다른 곳으로 분산됨.

---

## 0. 네트워크가 다루는 영역

네트워크는 **두 컴퓨터가 데이터를 교환** 하는 모든 과정을 다룬다:
- 물리 — 전기 신호, 광 신호, 무선 전파
- 링크 — 같은 매체 위의 노드 사이
- 망 — 서로 다른 네트워크 간 라우팅
- 종단 — 두 응용 사이의 신뢰 / 순서 / 흐름
- 응용 — HTTP / DNS / 메일 등 실제 사용

ARPANET 1969, TCP/IP 1973, WWW 1989, HTTP/3 (QUIC) 2022 까지의 누적이 오늘의 인터넷.

---

## 1. OSI 7 계층 — 깊이

[[osi-7-layer/osi-7-layer|↗ OSI 7 계층 hub]]

| 계층 | 전송 단위 | 노트 |
| --- | --- | --- |
| L1 Physical | bit | [[osi-7-layer/layer-1-physical/layer-1-physical]] |
| L2 Data Link | frame | [[osi-7-layer/layer-2-data-link/layer-2-data-link]] |
| L3 Network | packet | [[osi-7-layer/layer-3-network/layer-3-network]] |
| L4 Transport | segment | [[osi-7-layer/layer-4-transport/layer-4-transport]] |
| L5 Session | data | [[osi-7-layer/layer-5-session/layer-5-session]] |
| L6 Presentation | data | [[osi-7-layer/layer-6-presentation/layer-6-presentation]] |
| L7 Application | message | [[osi-7-layer/layer-7-application/layer-7-application]] |

---

## 2. 전송 계층 프로토콜 — 깊이

- [[tcp/tcp|↗ TCP]] — 3-way / 4-way / 슬라이딩 윈도우 / 혼잡 제어 (Slow Start / AIMD / CUBIC / BBR) / 상태 머신
- [[udp/udp|↗ UDP]] — 비연결, 헤더 8 byte, 응용
- [[quic/quic|↗ QUIC]] — UDP 위의 TCP + TLS 1.3 통합 (HTTP/3 의 기반)

---

## 3. 망 계층 — IP / 라우팅

- [[ip/ip|↗ IP hub]] — IPv4, IPv6, Subnetting, CIDR, NAT, ARP, ICMP, IGMP
- [[routing/routing|↗ 라우팅 hub]] — RIP, OSPF, BGP, EIGRP, MPLS

---

## 4. 응용 계층 프로토콜 — 깊이

- [[http/http|↗ HTTP hub]] — HTTP/1.0, 1.1, 2, 3 + 메서드 / 상태 코드 / 헤더 / 캐싱 / 쿠키 / CORS / REST / WebSocket
- [[tls-ssl/tls-ssl|↗ TLS / SSL]] — handshake, 인증서, mTLS, TLS 1.3
- [[dns/dns|↗ DNS]] — records, 계층, DNSSEC, DoH, DoT
- [[mail/mail|↗ 메일 프로토콜]] — SMTP, POP3, IMAP, SPF, DKIM, DMARC
- [[ssh-rdp-vpn/ssh-rdp-vpn|↗ 원격 / VPN]] — SSH, RDP, VPN (IPsec, WireGuard, OpenVPN)

---

## 5. 인프라 / 모던

- [[load-balancing/load-balancing|↗ Load Balancing]] — L4 vs L7, 알고리즘, HAProxy / Nginx / ALB / NLB
- [[cdn-proxy/cdn-proxy|↗ CDN / Proxy]] — Forward / Reverse Proxy, Edge cache, Cloudflare
- [[rpc-messaging/rpc-messaging|↗ RPC / Messaging]] — gRPC, GraphQL, WebSocket, MQTT, AMQP, WebRTC

---

## 6. 도구 / 실습

- [[tools/tools|↗ 네트워크 도구]] — ping / traceroute / dig / nslookup / tcpdump / Wireshark / curl / openssl / wrk / iperf

---

## 7. 토픽 / 면접

- [[topics/topics|↗ 토픽 hub]] — URL 입력부터 렌더링까지 / Fallacies / C10K / Zero Trust / Conway's Law / BGP 사건사

---

## 8. 핵심 키워드 한 줄

| 키워드 | 의미 |
| --- | --- |
| OSI 7 / TCP/IP 4 | 추상화 계층 모델 |
| MAC / IP | L2 / L3 주소 |
| TCP / UDP / QUIC | 종단 프로토콜 |
| HTTP/1/2/3 | 웹 응용 진화 |
| TLS 1.3 | 표준 암호화 |
| DNS | 이름 → IP |
| BGP | 인터넷 라우팅 글루 |
| CORS | 브라우저 도메인 정책 |
| Zero Trust | 네트워크 신뢰 X |
| QUIC | TCP + TLS 통합 |

---

## 관련

- [[../security-theory/security-theory]] — TLS, OAuth, 인증
- [[../operating-system/operating-system]] — Socket, epoll
- [[../distributed-systems/distributed-systems]] — 분산 통신
- [[../computer-science|↑ computer-science]]
