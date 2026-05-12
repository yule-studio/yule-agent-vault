---
title: "network 깊이 분리 — 진행 상황 (재개용)"
kind: progress
project: computer-science
agent: engineering-agent/tech-lead
status: in-progress
created_at: 2026-05-13T18:50:00+09:00
tags:
  - progress
  - network
  - resume-point
---

# network 깊이 분리 — 진행 상황 (재개용)

> 다음 세션에서 이 파일을 먼저 읽고 작업 재개. 사용자 의도: **오늘 안에 network 영역을 완전히 깊이 분리**.

---

## 작업 정책 (사용자 요구사항)

1. **사용자가 보여준 OSI 예시 수준의 깊이** — 단순 요약이 아닌 사전 수준 + 응용 + 함정
2. **각 개념 = 별도 파일** — 한 파일에 묶지 말고 세분화
3. **폴더 = 큰 영역** (예: tcp/, http/, dns/), 폴더 안에 hub + 세부 노트
4. **메타 표** 포함 — 전송 단위 / 대표 장치 / 대표 프로토콜
5. **함정 / 디버깅 / 면접 / 학습 자료** 항상 포함
6. **commit per 폴더 묶음** — 한 폴더 완성 후 commit + 본문에 노트 목록

---

## ✅ 완료 (45 노트)

### Hub
- [x] `network.md` — hub 재편 (단일 사전 → 폴더 인덱스)
- [x] `osi-7-layer/osi-7-layer.md` — 7 계층 hub

### L1 Physical (5)
- [x] `osi-7-layer/layer-1-physical/layer-1-physical.md`
- [x] `osi-7-layer/layer-1-physical/transmission-media.md`
- [x] `osi-7-layer/layer-1-physical/encoding-modulation.md`
- [x] `osi-7-layer/layer-1-physical/noise-and-errors.md`
- [x] `osi-7-layer/layer-1-physical/cables-connectors-devices.md`

### L2 Data Link (10)
- [x] `osi-7-layer/layer-2-data-link/layer-2-data-link.md`
- [x] `osi-7-layer/layer-2-data-link/ethernet-frame.md`
- [x] `osi-7-layer/layer-2-data-link/mac-address.md`
- [x] `osi-7-layer/layer-2-data-link/csma-cd-ca.md`
- [x] `osi-7-layer/layer-2-data-link/error-detection-crc.md`
- [x] `osi-7-layer/layer-2-data-link/flow-control.md`
- [x] `osi-7-layer/layer-2-data-link/switching-bridging.md`
- [x] `osi-7-layer/layer-2-data-link/vlan-stp-bonding.md`
- [x] `osi-7-layer/layer-2-data-link/dte-dce-interfaces.md`
- [x] `osi-7-layer/layer-2-data-link/wireless-link.md`

### L3 Network (7)
- [x] `osi-7-layer/layer-3-network/layer-3-network.md`
- [x] `osi-7-layer/layer-3-network/arp.md`
- [x] `osi-7-layer/layer-3-network/icmp.md`
- [x] `osi-7-layer/layer-3-network/fragmentation-mtu.md`
- [x] `osi-7-layer/layer-3-network/ip-routing-basics.md`
- [x] `osi-7-layer/layer-3-network/multicast-igmp.md`
- [x] `osi-7-layer/layer-3-network/ipsec-vpn.md`

### L4-L7 OSI hubs (4)
- [x] `osi-7-layer/layer-4-transport/layer-4-transport.md`
- [x] `osi-7-layer/layer-5-session/layer-5-session.md`
- [x] `osi-7-layer/layer-6-presentation/layer-6-presentation.md`
- [x] `osi-7-layer/layer-7-application/layer-7-application.md`

### tcp/ (11)
- [x] `tcp/tcp.md`
- [x] `tcp/tcp-header.md`
- [x] `tcp/three-way-handshake.md`
- [x] `tcp/four-way-termination.md`
- [x] `tcp/tcp-state-machine.md`
- [x] `tcp/sliding-window.md`
- [x] `tcp/congestion-control.md`
- [x] `tcp/tcp-options.md`
- [x] `tcp/tcp-keepalive.md`
- [x] `tcp/tcp-performance-tuning.md`
- [x] `tcp/tcp-attacks.md`

### udp/ (3)
- [x] `udp/udp.md`
- [x] `udp/udp-vs-tcp.md`
- [x] `udp/udp-applications.md`

### quic/ (3)
- [x] `quic/quic.md`
- [x] `quic/quic-handshake.md`
- [x] `quic/quic-streams.md`

---

## ⏳ 남은 작업 (예상 60+ 노트)

### Phase A — 망 (재개 시 1순위)

#### ip/ (목표 6 노트)
- [ ] `ip/ip.md` — hub
- [ ] `ip/ipv4-header.md`
- [ ] `ip/ipv6-header.md`
- [ ] `ip/cidr-subnetting.md`
- [ ] `ip/nat.md` — NAT44 / NAT64 / CGNAT / Hairpinning
- [ ] `ip/ipv4-vs-ipv6.md`

#### routing/ (목표 7 노트)
- [ ] `routing/routing.md` — hub
- [ ] `routing/rip.md`
- [ ] `routing/ospf.md`
- [ ] `routing/bgp.md`
- [ ] `routing/eigrp.md`
- [ ] `routing/mpls.md`
- [ ] `routing/pim-multicast.md`

### Phase B — 응용 (재개 시 2순위 — 가장 자주 쓰는)

#### http/ (목표 11 노트)
- [ ] `http/http.md` — hub
- [ ] `http/http-1-0.md`
- [ ] `http/http-1-1.md`
- [ ] `http/http-2.md`
- [ ] `http/http3-quic.md`
- [ ] `http/methods.md`
- [ ] `http/status-codes.md`
- [ ] `http/headers.md`
- [ ] `http/caching.md`
- [ ] `http/cookies.md`
- [ ] `http/cors.md`
- [ ] `http/rest-api-design.md`
- [ ] `http/websocket.md` (또는 rpc-messaging 으로 이동)

#### tls-ssl/ (목표 6 노트)
- [ ] `tls-ssl/tls-ssl.md` — hub
- [ ] `tls-ssl/tls-handshake-1-2.md`
- [ ] `tls-ssl/tls-1-3.md`
- [ ] `tls-ssl/certificates-ca.md`
- [ ] `tls-ssl/mtls.md`
- [ ] `tls-ssl/cipher-suites.md`

#### dns/ (목표 6 노트)
- [ ] `dns/dns.md` — hub
- [ ] `dns/dns-hierarchy-resolution.md`
- [ ] `dns/dns-record-types.md`
- [ ] `dns/dnssec.md`
- [ ] `dns/doh-dot.md`
- [ ] `dns/dns-caching.md`

#### mail/ (목표 5 노트)
- [ ] `mail/mail.md` — hub
- [ ] `mail/smtp.md`
- [ ] `mail/pop3.md`
- [ ] `mail/imap.md`
- [ ] `mail/spf-dkim-dmarc.md`

#### ssh-rdp-vpn/ (목표 4 노트)
- [ ] `ssh-rdp-vpn/ssh-rdp-vpn.md` — hub
- [ ] `ssh-rdp-vpn/ssh.md`
- [ ] `ssh-rdp-vpn/rdp.md`
- [ ] `ssh-rdp-vpn/vpn-types.md` (IPsec/OpenVPN/WireGuard 통합)

### Phase C — 인프라 (3순위)

#### load-balancing/ (목표 4 노트)
- [ ] `load-balancing/load-balancing.md` — hub
- [ ] `load-balancing/l4-vs-l7.md`
- [ ] `load-balancing/algorithms.md` — round-robin / least-conn / consistent-hash / hash
- [ ] `load-balancing/sticky-session.md`

#### cdn-proxy/ (목표 4 노트)
- [ ] `cdn-proxy/cdn-proxy.md` — hub
- [ ] `cdn-proxy/forward-proxy.md`
- [ ] `cdn-proxy/reverse-proxy.md` (Nginx, Envoy)
- [ ] `cdn-proxy/cdn.md`

#### rpc-messaging/ (목표 6 노트)
- [ ] `rpc-messaging/rpc-messaging.md` — hub
- [ ] `rpc-messaging/grpc.md`
- [ ] `rpc-messaging/graphql.md`
- [ ] `rpc-messaging/websocket.md`
- [ ] `rpc-messaging/mqtt-amqp.md`
- [ ] `rpc-messaging/webrtc.md`

### Phase D — 도구 / 토픽 (4순위)

#### tools/ (목표 6 노트)
- [ ] `tools/tools.md` — hub
- [ ] `tools/ping-traceroute-mtr.md`
- [ ] `tools/netstat-ss.md`
- [ ] `tools/tcpdump-wireshark.md`
- [ ] `tools/dig-nslookup-host.md`
- [ ] `tools/curl-openssl.md`
- [ ] `tools/wrk-iperf.md`

#### topics/ (목표 5 노트)
- [ ] `topics/topics.md` — hub
- [ ] `topics/url-to-rendering.md` — "주소창 입력하면 무슨 일?"
- [ ] `topics/c10k-c10m.md`
- [ ] `topics/zero-trust.md`
- [ ] `topics/bgp-incidents.md` — Facebook 2021 등
- [ ] `topics/bufferbloat.md`

---

## 재개 절차

다음 세션 시작 시:

1. 이 파일 (`_progress.md`) 먼저 읽기
2. **Phase A → B → C → D** 순서로
3. 각 폴더 작성 후 `git commit` (본문에 노트 목록)
4. Phase 끝나면 `git push` + 진행 상황 (`_progress.md`) 갱신
5. 사용자에게 짧은 보고 (현재 / 다음)

---

## 작성 표준 (각 노트가 따라야 함)

```markdown
---
title: "..."
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T...:00+09:00
tags: [...]
---

# 제목

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | ... |

**[[상위 hub]]** · **[[조상 hub]]**

## (메타 표) — 해당 시
| 전송 단위 | 대표 장치 | 대표 프로토콜 |

## 1. 한 줄 정의

## 2. 역사 / 배경 (해당 시)

## 3. 핵심 개념 / 동작 (그림 + 표)

## 4. 변형 / 응용

## 5. 함정 / 안티패턴

## 6. 도구 / 디버깅 (해당 시)

## 7. 면접 / 토픽

## 8. 학습 자료

## 9. 관련
```

각 노트는 **사용자 OSI 예시 수준 깊이** — 단순 정의가 아닌 실무 + 함정 + 표.

---

## 메모리 위치 (claude 메모리)

- `project_network_deep_split.md` — 진행 상황 + 정책

다음 세션에 메모리에서 자동 로드 + 이 파일 추가 확인.
