---
title: "Bufferbloat / BBR / AQM"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:25:00+09:00
tags:
  - network
  - performance
  - bufferbloat
  - tcp
---

# Bufferbloat / BBR / AQM

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 큰 버퍼의 latency 폭증 |

**[[topics|↑ 토픽 hub]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

**라우터 / 디바이스의 큰 버퍼** 가 TCP congestion control 을 망쳐 — 대역 만족 시 latency 폭증.

---

## 2. 문제

### 시나리오
```
가정 라우터 — 100 ms 버퍼
사용자 — bulk upload (Zoom upload 등)
        ↓
TCP 가 큰 버퍼 채움 → ACK 늦음 → cwnd 계속 ↑
        ↓
다른 패킷 (ping / VoIP) — 100 ms 대기
```

### 결과
- ping 10 ms → 200 ms
- Zoom 끊김 / 음성 떨림
- 게임 / 실시간 망

---

## 3. 원인

### 큰 버퍼 (memory 싸짐)
- 옛 — 64 KB
- 모던 라우터 — MB 단위

### TCP 의 옛 congestion control
- **CUBIC** (Linux 기본 옛) — loss 기반
- 패킷 loss 까지 cwnd ↑
- 큰 버퍼 — loss 없이 latency 만 ↑

### 잘못된 가정
- "버퍼 = 안전" (loss 방지) — 실제 latency 의 적

---

## 4. Jim Gettys (2010) — 명명

### 발견
- 2010 — Jim Gettys (Bell Labs) bufferbloat.net
- 가정 / WiFi 라우터의 큰 영향

### 인식
- "ICTF" — Internet Connectivity Test
- DSLReports — bufferbloat test
- Speedtest — bufferbloat score

---

## 5. 해결 — BBR (Google, 2016)

### Bottleneck Bandwidth and RTT
- 손실이 아닌 **RTT + bandwidth** 기반
- 큐 채우기 안 함 → low latency

### 동작
```
1. probe bandwidth — 최대 bw 측정
2. probe RTT — 최소 RTT 측정
3. cwnd = bw × RTT_min (BDP)
```

### 효과
- Bufferbloat 없음
- Loss 신호 없이 — flow 안정

### 도입
- Linux kernel (4.9+)
- Google services
- YouTube / Spotify / Netflix

---

## 6. AQM — Active Queue Management

### 정의
- 라우터가 적극적으로 buffer 관리
- 큐 채우기 전 — 패킷 drop / mark (ECN)

### CoDel (Controlled Delay) — 2012
- Van Jacobson + Kathleen Nichols
- "Sojourn time" — 패킷이 큐에서 대기한 시간
- Threshold (5 ms) — drop

### fq_codel — Flow Queue CoDel
- Flow 별 queue
- 각 flow 별 codel
- WiFi / 가정 router 친화

### PIE — Proportional Integral controller Enhanced
- Cisco — 비슷

### CAKE — Common Applications Kept Enhanced
- fq_codel + shaping + 잡다한 기능
- OpenWRT / 가정 라우터

---

## 7. ECN — Explicit Congestion Notification

### 정의
- 라우터가 IP 헤더에 마크 (drop 대신)
- TCP — 마크 보고 cwnd 줄임
- Loss 없이 congestion 신호

### 효과
- Loss / retransmission ↓
- Latency ↓

### 도입
- 옛 — block 하는 middlebox
- 모던 — 점진

---

## 8. L4S — Low Latency, Low Loss, Scalable

### 정의
- 2020s — 모던 표준
- ECN + scalable congestion control
- 1 ms 단위 latency

### 도구
- Dual-queue (low-latency / classic)
- DCTCP / TCP Prague

### 도입
- Apple 시작
- Comcast 일부

---

## 9. 측정

### DSLReports
- https://www.dslreports.com/speedtest
- Bufferbloat score (A / B / C / D / F)

### waveform.com/tools/bufferbloat
- 모던 도구

### fast.com
- Netflix — 단순 throughput

### 자체 측정
```bash
# 기준 ping
ping -c 100 example.com

# 부하 시 ping
# 다른 터미널 — upload (iperf3 -c)
ping -c 100 example.com
# RTT 증가 보면 — bufferbloat
```

---

## 10. 해결 — 사용자 / 운영자

### 라우터 설정
- OpenWRT — sqm-scripts (CAKE)
- pfsense — fq_codel
- 일부 ISP 라우터 — 자동

### Linux 시스템
```bash
# 기본 qdisc 변경
sudo sysctl net.core.default_qdisc=fq_codel
# 또는 cake

# Congestion control
sudo sysctl net.ipv4.tcp_congestion_control=bbr
# 옵션: cubic / reno / bbr / bbr2
```

### 라우터 — buffer 줄이기
- 무의미 — buffer 줄여도 다른 문제
- AQM 권장

---

## 11. WiFi 의 특수

### 옛 WiFi (a/b/g/n)
- 단일 큐 — 한 packet 의 retransmission 모두 막힘
- 큰 buffer 별로 — bufferbloat 심함

### WiFi 6 (802.11ax)
- 모던 — fq_codel 도입
- 다중 stream / OFDMA

---

## 12. 셀룰러 (4G/5G)

### 큰 버퍼 — 더 심함
- 옛 — 수 초 buffer (DSL/dial-up 가정)
- 4G/5G — 부하 시 latency 폭증

### 5G — 개선
- L4S 도입
- Slice 별 우선순위

---

## 13. CDN / 인터넷 backbone

### 모던
- BBR / BBRv2
- ECN
- 영향 적음

### 옛 transit ISP
- CUBIC + 큰 buffer
- Peering 시점 bufferbloat

---

## 14. 함정

### 함정 1 — 빠른 인터넷 ≠ 빠른 latency
1 Gbps 도 — bufferbloat 시 ping 200 ms.

### 함정 2 — 라우터 cheap
저렴 라우터 — 단일 큐 + 큰 buffer. 의도된 bufferbloat.

### 함정 3 — TCP test 만 보면 모름
Speedtest — throughput 만. 실제 — ping under load.

### 함정 4 — BBR 의 fairness
초기 BBR — CUBIC 과 같이 있을 때 BBR 우세 (CUBIC 불리).
BBRv2 — 개선.

### 함정 5 — ECN 의 미들박스
일부 옛 라우터 — ECN bit 무시 / 변경. 모던 — OK.

### 함정 6 — 사용자 측 해결의 한계
ISP 의 라우터가 bloated — 사용자 해결 불가.

---

## 15. 학습 자료

- bufferbloat.net
- "Bufferbloat" (Jim Gettys, IEEE)
- "BBR" (Google paper, 2016)
- "CoDel" — RFC 8289
- "L4S" — IETF docs

---

## 16. 관련

- [[topics]] — Hub
- [[../tcp/tcp-congestion-control]] (TBD) — BBR / CUBIC
- [[../tcp/tcp]] — TCP buffer
