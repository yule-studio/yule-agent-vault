---
title: "TCP 공격 (SYN Flood / RST / Sequence Prediction / Hijacking)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:10:00+09:00
tags:
  - network
  - tcp
  - security
  - syn-flood
---

# TCP 공격 (SYN Flood / RST / Sequence Prediction / Hijacking)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | SYN Flood / RST / Hijack / Slowloris / Reflection / TCP MD5 |

**[[tcp|↑ TCP]]** · **[[../network|↑↑ network hub]]**

---

## 1. SYN Flood

### 1.1 공격

```
공격자 → SYN, spoofed Src IP × N
서버: 각 SYN 에 TCB 할당, SYN+ACK 응답 → ACK 안 옴
→ SYN_RECEIVED 누적 → 자원 소진
```

자세히 → [[three-way-handshake#5 SYN Flood Attack]]

### 1.2 방어
- **SYN Cookie** — TCB 할당 없이 stateless
- **Backlog 증가**
- **Rate limiting** — IP 별
- **First SYN drop** — 진짜 클라는 재시도
- **SYN proxy** — 방화벽 / LB 가 대신
- **Anycast + 분산** — 부하 분산

---

## 2. RST Attack (Reset Injection)

### 2.1 공격

```
공격자: 두 호스트 간 ESTABLISHED 연결 탐지 (Src/Dst IP, Port)
공격자: RST 패킷 위조 — Seq# 추측 또는 윈도우 내 모든 Seq# 시도
서버 / 클라: RST 수신 → 즉시 연결 종료
```

### 2.2 1985 Mitnick / Morris 공격
- 초기 ISN 이 timer 기반 → 추측 가능
- RST 또는 데이터 주입으로 연결 hijack

### 2.3 방어
- **랜덤 ISN** (RFC 6528) — 모던 OS
- **TCP-MD5 / TCP-AO** — BGP 등에 인증
- **윈도우 외 RST 거부** — RFC 5961
- **IPsec / TLS** — 응용 보호

---

## 3. Sequence Prediction Attack

### 3.1 공격
- 초기 ISN 패턴 분석 → 다음 ISN 추측
- A 와 B 의 연결에 위조 패킷 주입 → 데이터 변조 / Hijack

### 3.2 1995 Mitnick 의 Shimomura 침입
- 약한 ISN 의 SunOS 4.x
- 신뢰 IP 위조 + 추측한 Seq#
- 유명 침입 사례

### 3.3 방어
- 강한 랜덤 ISN (RFC 1948, RFC 6528)
- IPsec / TLS

---

## 4. TCP Hijacking

### 4.1 정의
이미 수립된 연결에 끼어들어 변조.

### 4.2 방법
- MITM (ARP Spoofing 등)
- Sequence prediction + injection
- IP spoofing + 충분한 정보

### 4.3 방어
- **TLS** — 모든 응용 데이터 암호화
- **SSH** — 원격 접속
- **mTLS** — 양방향 인증
- **IPsec** — L3 레벨 암호화

---

## 5. Slowloris (Slow HTTP)

### 5.1 공격

```
공격자: HTTP 요청을 천천히 보냄
        - Headers 전체 안 보내기
        - 또는 매우 천천히 (1 byte / 분)
서버: keep-alive timeout 안 도달 → 연결 유지 → 자원 소진
```

TCP 자체 X — HTTP / 응용 레벨이지만 TCP 의 keep-alive 활용.

### 5.2 방어
- Connection limit per IP
- Header / request timeout (Nginx `client_header_timeout`)
- mod_reqtimeout (Apache)
- WAF / Cloudflare

---

## 6. SYN-ACK / RST Reflection (Amplification)

### 6.1 공격

```
공격자 → SYN (Src=Victim) → 서버 1
공격자 → SYN (Src=Victim) → 서버 2
...
각 서버 → SYN+ACK → Victim
Victim: 의도 X SYN+ACK 폭격
```

DRDoS (Distributed Reflection DoS).

### 6.2 증폭률
- TCP 의 SYN+ACK 증폭은 작음 (1:1 또는 미세)
- 더 큰 증폭: DNS (1:50), NTP (1:200), Memcached (1:51000)

### 6.3 방어
- **BCP 38** — ISP 의 source IP filtering (Egress filter)
- Anycast / Scrubbing 서비스 (Cloudflare)

---

## 7. ACK Flood

대량 ACK → 서버 자원 소진. 보통 SYN Flood 와 함께.

### 방어
- Stateful 방화벽 — 모르는 connection 의 ACK 폐기
- ACK rate limit

---

## 8. Connection Exhaustion

### 8.1 공격
대량 정상 연결 수립 후 idle 유지 → 서버 connection 한계 도달.

### 8.2 방어
- per-IP connection limit
- timeout 짧게
- keepalive 단축

---

## 9. TCP Window Scale 부재 공격 (옛)

- 미들박스가 Option 제거 → 작은 Window
- 의도적 stripping 으로 성능 저하

→ 모던 미들박스 대부분 OK.

---

## 10. Optimistic ACK (ROACH)

### 10.1 공격
- 아직 받지 않은 segment 의 ACK 미리 보내기
- 송신자가 cwnd 빠르게 증가 → 폭증 → 네트워크 마비

### 10.2 방어
- Sender 가 ACK 일관성 검사 (의심스러우면 무시)
- 잘 알려지지 않은 공격

---

## 11. TCP Reset 의 합법적 사용 vs 공격

### 11.1 합법적 RST
- 닫힌 포트로 SYN → RST
- 응용 크래시 시 OS 가 RST
- TIME_WAIT 단축 (SO_LINGER 0)

### 11.2 부정 사용
- ISP / 정부가 검열용 RST 주입 (중국 방화벽, 영국 BT 등)
- 워크어라운드: TLS / VPN / Tor

---

## 12. 보호 기술 종합

### 12.1 TCP 레벨

- **SYN Cookie**
- **Random ISN** (RFC 6528)
- **RFC 5961** — Window 안의 RST 만 수용
- **TCP-MD5** (BGP) / **TCP-AO** (RFC 5925)
- **PAWS** (Timestamp) — wrap-around 보호

### 12.2 응용 / 시스템 레벨

- **TLS** — 응용 데이터 암호화
- **IPsec** — L3 캡슐화
- **Rate limit** — 응용 / 방화벽
- **WAF** — Slowloris 등
- **Anti-DDoS** — Cloudflare, Akamai
- **BCP 38** — Spoofing 차단

---

## 13. 방화벽 / IDS 가 보는 신호

- 대량 SYN — SYN Flood
- 대량 ACK / RST — flood or reset attack
- 작은 window size — Slowloris
- 비정상 옵션 조합 — Fingerprinting / OS detect
- 빠른 연결 시도 — Port scan (`nmap`)

---

## 14. Linux 의 방어 sysctl

```bash
# SYN cookie
net.ipv4.tcp_syncookies = 1

# 약한 ISN 방지 (이미 기본)
net.ipv4.tcp_secure_rfc6528 = 1   # 일부 커널

# RFC 5961 (Reset/SYN/Data injection 방어)
# Linux 3.6+ 자동

# Connection per IP 제한
iptables -A INPUT -p tcp --syn --dport 80 -m connlimit \
  --connlimit-above 50 -j REJECT

# SYN rate limit
iptables -A INPUT -p tcp --syn --dport 80 -m limit \
  --limit 100/s --limit-burst 200 -j ACCEPT
iptables -A INPUT -p tcp --syn --dport 80 -j DROP
```

---

## 15. 학습 자료

- RFC 4987 (SYN flood 방어), RFC 5961 (RST/SYN 주입 방어)
- RFC 6528 (Random ISN)
- "TCP Reset Injection" — Sandvine / Comcast 사례
- CERT — Slowloris advisory
- Cloudflare "Understanding TCP DoS"

---

## 16. 관련

- [[three-way-handshake]] — SYN Flood
- [[four-way-termination]] — RST 비정상 종료
- [[tcp-options]] — TCP-MD5, TCP-AO
- [[../../security-theory/security-theory]] — DDoS 일반
- [[tcp]] — TCP hub
