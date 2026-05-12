---
title: "NAT — Network Address Translation"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:15:00+09:00
tags:
  - network
  - nat
  - cgnat
  - port-forwarding
---

# NAT — Network Address Translation

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | SNAT/DNAT/PAT, CGNAT, NAT64, Hairpinning, NAT Traversal |

**[[ip|↑ IP]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

**IP 주소 / 포트 를 변환** — 주로 사설 (RFC 1918) ↔ 공인 IP 변환. IPv4 부족의
임시방편 (RFC 3022, 2001).

---

## 2. 왜 NAT 인가

### 2.1 IPv4 부족
- 32 bit = ~43 억 — 1990 년대부터 부족 예상
- 1990: PAT (Port Address Translation) 발명
- 2000: 사실상 모든 가정 / 회사가 NAT
- 2011: IANA 의 IPv4 풀 소진

### 2.2 부가 효과
- 보안 (외부에서 내부로 직접 접근 X)
- IP 구조 숨김
- 비용 절약 (공인 IP 1 개로 수천 호스트)

### 2.3 단점
- **End-to-end principle 깨짐**
- P2P 어려움 (WebRTC 의 NAT Traversal)
- 일부 프로토콜 (IPsec ESP, GRE) 통과 어려움
- 응용 layer 가 IP 의식하면 깨짐 (FTP active mode)

---

## 3. NAT 의 4 종류

### 3.1 Static NAT (1:1)

```
사설 10.0.0.5  ↔  공인 203.0.113.5  (영구 매핑)
```

- 서버 외부 공개에 사용
- IP 절약 X

### 3.2 Dynamic NAT (Pool, 1:1)

```
사설 10.0.0.0/24  →  공인 pool [203.0.113.1-10]
호스트마다 동적 할당
```

- 동시 접속 수 = pool 크기 한계

### 3.3 PAT — Port Address Translation (1:N, "NAPT")

가장 흔함 — 가정용 라우터의 NAT.

```
내부:
10.0.0.5:50000 → google.com:443
10.0.0.6:50001 → google.com:443
10.0.0.7:50002 → google.com:443

NAT Table (공인 IP 1.2.3.4):
1.2.3.4:30000  ↔  10.0.0.5:50000
1.2.3.4:30001  ↔  10.0.0.6:50001
1.2.3.4:30002  ↔  10.0.0.7:50002

→ 한 공인 IP 로 수만 호스트 (65535 port 한계)
```

### 3.4 CGNAT — Carrier-Grade NAT

```
공인 IP        ←  CGNAT  ←  Carrier NAT  ←  사설 (100.64.0.0/10)
ISP 운영                                       ↑
                                          가정 라우터 NAT
                                          (192.168.1.0/24)
```

→ **이중 NAT** — 모바일 / 일부 ISP. P2P 거의 불가능.

---

## 4. SNAT vs DNAT

### 4.1 SNAT — Source NAT
- **출발 IP / Port 변경**
- 내부 → 외부 갈 때
- "Masquerade" (Linux 의 dynamic SNAT)

```
10.0.0.5:50000 → 8.8.8.8:80
     ↓ SNAT
1.2.3.4:30000 → 8.8.8.8:80
```

### 4.2 DNAT — Destination NAT
- **목적지 IP / Port 변경**
- 외부 → 내부 (Port Forwarding)
- 서버 공개

```
인터넷 → 1.2.3.4:80
     ↓ DNAT
인터넷 → 10.0.0.5:8080
```

### 4.3 Linux iptables 예

```bash
# SNAT (Masquerade) — 모든 outbound
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
# 또는 명시
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 \
  -j SNAT --to-source 1.2.3.4

# DNAT (Port Forwarding)
iptables -t nat -A PREROUTING -p tcp -d 1.2.3.4 --dport 80 \
  -j DNAT --to-destination 10.0.0.5:8080
```

---

## 5. NAT Table (Connection Tracking)

NAT 는 **stateful** — 매 흐름 추적:

```
+--------------+--------------+-----------+------+
| Inside (IP:Port) | Outside (IP:Port) | Protocol | TTL |
+--------------+--------------+-----------+------+
| 10.0.0.5:50000 | 8.8.8.8:80 | TCP | 7200 |
| 10.0.0.6:50001 | 1.1.1.1:443 | TCP | 7200 |
| ...           | ...          | ...       | ...  |
+--------------+--------------+-----------+------+
```

### 타임아웃
- **TCP** — 2 시간 (idle), 1 분 (FIN 후)
- **UDP** — 30 초 - 5 분
- **ICMP** — 30 초

→ **UDP keepalive 필수** (모바일 NAT).

### Linux 의 conntrack

```bash
sudo conntrack -L                 # 전체
sudo conntrack -L -p tcp           # TCP 만
sudo conntrack -E                  # 이벤트 stream
cat /proc/sys/net/netfilter/nf_conntrack_max
cat /proc/sys/net/netfilter/nf_conntrack_count
```

---

## 6. NAT 의 종류 (RFC 3489) — STUN 분류

P2P 의 영향:

### 6.1 Full Cone
```
내부 (IP:port) → NAT → 외부 (IP:port)
이 매핑이 만들어지면:
- 어떤 외부에서든 외부 (IP:port) 로 보내면 내부 도달
```
가장 관대. P2P 쉬움.

### 6.2 Restricted Cone
```
내부 → NAT → 외부 (IP:port)
"보낸 적 있는 외부 IP" 에서 온 패킷만 받음 (포트 무관)
```

### 6.3 Port-Restricted Cone
```
"보낸 적 있는 외부 IP:port" 의 정확한 응답만
```

### 6.4 Symmetric NAT — 최악
```
같은 내부 → 다른 외부 = 다른 외부 매핑

10.0.0.5:50000 → 8.8.8.8 → 1.2.3.4:30000
10.0.0.5:50000 → 1.1.1.1 → 1.2.3.4:30001    ← 다른 포트!
```

→ STUN 정보로 P2P 시도해도 안 통함. **TURN relay** 필요.

---

## 7. NAT Traversal — P2P 가능하게

### 7.1 STUN (RFC 5389)

```
1. 내부 호스트 → STUN 서버 (인터넷)
   "내 외부 IP/Port 알려줘"
2. STUN 서버 → 호스트
   "너는 1.2.3.4:30000 으로 보이네"
3. 두 호스트가 STUN 정보 교환 (별도 서버)
4. 직접 통신 시도 (Hole Punching)
```

### 7.2 TURN (RFC 5766)
- STUN 실패 시 (Symmetric NAT)
- 중계 서버가 패킷 릴레이
- 대역폭 비쌈

### 7.3 ICE (RFC 8445)
- STUN + TURN 의 통합
- WebRTC 의 표준

### 7.4 UPnP / NAT-PMP / PCP
- 라우터에게 "포트 열어줘"
- 자동 Port Forwarding
- 보안 위험 — 비활성 권장

---

## 8. Hairpinning (NAT Loopback)

```
내부 → 자기 공인 IP → DNAT → 자기 내부 서버

문제: 일부 NAT 가 같은 인터페이스로 들어와 나가는 트래픽 차단
```

### 해결
- 모던 라우터는 hairpinning 지원
- 또는 split-horizon DNS (내부에서는 내부 IP 응답)

---

## 9. CGNAT 의 영향

- IP 별 사용자 식별 어려움 (수천 사용자가 같은 IP)
- 외부 서비스의 IP 차단 시 무고한 사용자 영향
- 게임 / VoIP / P2P 어려움 → IPv6 권장

---

## 10. NAT64 — IPv6 클라이언트 → IPv4 서버

```
IPv6 클라이언트 ─→ NAT64 ─→ IPv4 서버
2001:db8::1       64:ff9b::8.8.8.8     8.8.8.8
```

표준 prefix `64:ff9b::/96` (RFC 6052).

### DNS64 (RFC 6147)
- DNS resolver 가 IPv4 의 A 레코드를 IPv6 AAAA 로 변환 (위 prefix 추가)
- 클라가 IPv6 만 보내도 IPv4 서버 접근

---

## 11. NPTv6 (RFC 6296)

IPv6 의 prefix 변환 (NAT 아닌):
- IP 자체는 보존 (호스트 부분)
- Prefix 만 1:1 변환
- IPv6 의 사설 / SLAAC 충돌 회피

진정한 IPv6 NAT 는 없음 — NPTv6 가 가장 가깝다.

---

## 12. NAT 의 문제 프로토콜

### 12.1 FTP Active Mode
- 클라가 자기 IP / Port 데이터 채널로 알림 → NAT 가 변경 안 함 → 깨짐
- 해결: ALG (Application Level Gateway) 가 FTP 명령 파싱하고 변환
- 또는 Passive Mode 사용

### 12.2 SIP / RTP (VoIP)
- 메시지 안의 IP 가 사설
- 해결: SIP ALG, STUN, ICE

### 12.3 IPsec ESP
- 포트 없음 → NAT 가 추적 어려움
- 해결: NAT-T (UDP 4500 캡슐화)

### 12.4 GRE
- IP Protocol 47 — 포트 없음
- 일부 NAT 지원 (PPTP)

---

## 13. 도구 / 디버깅

```bash
# 외부 IP 확인
curl ifconfig.me
curl ipinfo.io/ip

# NAT 종류 추측 (STUN)
# stun.l.google.com:19302

# Linux conntrack
sudo conntrack -L | head
sudo conntrack -F   # flush
```

---

## 14. 함정

### 함정 1 — NAT = 보안 가정
NAT 는 부수효과로 보호 — 진짜 방화벽 필요. 잘못 설정된 UPnP 도 위험.

### 함정 2 — UDP keepalive 누락
NAT timeout 30 초 → 흐름 끊김. 모바일에서 특히.

### 함정 3 — 64K 포트 한계
한 공인 IP 의 PAT 가 64K 흐름만. 큰 사용자 ↔ 한 도메인 다수 연결 시 부족.

### 함정 4 — Symmetric NAT 의 P2P
TURN 없으면 안 됨. WebRTC 의 5-10% 가 TURN 필요.

### 함정 5 — Hairpinning 미지원
같은 LAN 의 두 호스트가 공인 IP 로 서로 못 봄.

### 함정 6 — IPv6 의 NAT 가정
IPv6 는 보통 NAT X — 호스트 직접 노출. 방화벽 활성 필수.

### 함정 7 — Connection tracking 한계
`nf_conntrack_max` 가득 차면 새 연결 폐기 — 큰 NAT 게이트웨이에서 발생.

---

## 15. 학습 자료

- RFC 3022 (NAT), RFC 6888 (CGNAT), RFC 6146 (NAT64), RFC 5389 (STUN)
- **TCP/IP Illustrated Vol.1** Ch. 7
- "P2P NAT Traversal" — Bryan Ford
- Cloudflare blog — NAT traversal 시리즈

---

## 16. 관련

- [[ip]] — IP hub
- [[cidr-subnetting]] — 사설 IP 범위
- [[ipv6-header]] — NPTv6
- [[../rpc-messaging/rpc-messaging]] — WebRTC + STUN/TURN/ICE
