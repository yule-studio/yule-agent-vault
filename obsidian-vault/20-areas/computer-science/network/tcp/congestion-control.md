---
title: "TCP 혼잡 제어 (Congestion Control)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T17:50:00+09:00
tags:
  - network
  - tcp
  - congestion-control
  - cubic
  - bbr
---

# TCP 혼잡 제어 (Congestion Control)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Slow Start / AIMD / Tahoe / Reno / NewReno / CUBIC / BBR / ECN |

**[[tcp|↑ TCP]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

**네트워크의 혼잡 (라우터 큐 가득) 을 감지** 해 송신 속도 조절. 1986 NSFNET 붕괴
이후 Van Jacobson 1988 도입 — 인터넷이 살아남은 이유.

---

## 2. 1986 사건 — 왜 필요한가

```
1986 가을:
NSFNET 의 처리량이 32 kbps → 40 bps 로 폭락 (1000 배)
원인: 모든 호스트가 끊임없이 재전송 → 혼잡 → 더 큰 손실 → 더 많은 재전송
"Congestion Collapse"
```

1988 Van Jacobson "Congestion Avoidance and Control" 논문 — Slow Start + Congestion Avoidance.

---

## 3. 흐름 제어 vs 혼잡 제어 (재확인)

| 구분 | Flow | Congestion |
| --- | --- | --- |
| 보호 | 수신자 buffer | 네트워크 |
| 윈도우 | rwnd (수신자 광고) | cwnd (송신자 계산) |
| 신호 | Window 필드 | Loss / ECN / RTT |

**송신 가능 = min(rwnd, cwnd)**

---

## 4. 4 가지 상태 (RFC 5681)

### 4.1 Slow Start (느린 시작)

```
초기 cwnd = IW (Initial Window, 보통 10 MSS)
ACK 받을 때마다 cwnd += 1 MSS
→ 매 RTT 에 cwnd × 2 (Exponential growth)
```

이름은 "Slow" 지만 실제로는 빠른 (지수) 증가 — IW 가 작아서 "slow start".

### 4.2 Congestion Avoidance

ssthresh (slow start threshold) 도달 → 선형 증가:
```
cwnd += MSS × MSS / cwnd  (per ACK)
→ 매 RTT 에 cwnd += 1 MSS  (Linear)
```

### 4.3 Fast Retransmit

3 중복 ACK 받으면 즉시 재전송 (RTO 안 기다림):
```
seq 1000 → ACK 1001
seq 1500 (1500 손실, but 2000 받음)
seq 2000 → ACK 1500  (DUP)
seq 2500 → ACK 1500  (DUP 2)
seq 3000 → ACK 1500  (DUP 3) → 즉시 1500 재전송
```

### 4.4 Fast Recovery

3 DUP ACK 후:
- ssthresh = cwnd / 2
- cwnd = ssthresh (TCP Reno)
- Congestion Avoidance 로 진입

(RTO timeout 시 → Slow Start 부터 시작 — 더 강한 반응)

---

## 5. AIMD — Additive Increase, Multiplicative Decrease

- **AI** — 매 RTT 에 +1
- **MD** — 손실 시 ÷2

→ 톱니파 그래프:
```
cwnd
  ▲
  │       ╱│
  │    ╱   │   ╱│
  │ ╱       ╲╱  │
  ├─────────────│──→ time
  │            손실
```

이 알고리즘이 공정 (fairness) + 안정.

---

## 6. 알고리즘 진화

### 6.1 TCP Tahoe (1988)
- Slow Start + Congestion Avoidance + Fast Retransmit
- 손실 시: cwnd=1, Slow Start 부터

### 6.2 TCP Reno (1990)
- Tahoe + **Fast Recovery**
- 손실 시: cwnd=cwnd/2 (Slow Start X)
- 한 RTT 에 한 손실만 잘 처리

### 6.3 NewReno (RFC 2582, 1999)
- 한 윈도우의 여러 손실 처리
- Reno 보다 강함

### 6.4 SACK (RFC 2018)
- Selective ACK Option
- 어느 segment 받았는지 명시 → 선택 재전송 효율 ↑

### 6.5 Vegas (1994)
- RTT 변화로 혼잡 감지 (loss 전에)
- Latency-based
- Reno 와 공존 시 손해

### 6.6 BIC / CUBIC (2008)
- **CUBIC** — Linux 2.6.19+ 기본
- 3 차 함수로 cwnd 증가
- 큰 BDP 친화
- 손실 후 빠르게 회복

```
cwnd(t) = C × (t - K)³ + W_max
W_max = 손실 직전 cwnd
K = (W_max × β / C)^(1/3)
```

### 6.7 Compound TCP (CTCP, Windows Vista+)
- Reno + delay 컴포넌트
- 양쪽 신호 활용

### 6.8 BBR — Bottleneck Bandwidth + RTT (Google 2016)
- Loss 가 아닌 **대역폭 + 최소 RTT** 측정
- 4 단계: STARTUP → DRAIN → PROBE_BW → PROBE_RTT
- **Bufferbloat** 환경에서 큰 개선
- 단점: AIMD 와 공정성 문제 (BBRv2 가 개선)

### 6.9 BBRv2 / BBRv3
- 손실 / ECN 도 고려
- 공정성 개선

---

## 7. CUBIC 자세히

```python
# 의사 코드
def cubic_window(t, W_max, t_loss):
    C = 0.4
    beta = 0.7
    K = ((W_max * (1 - beta)) / C) ** (1/3)
    cwnd = C * ((t - t_loss - K) ** 3) + W_max
    return cwnd
```

특징:
- 손실 직후 빠른 회복 (K 까지)
- W_max 근처에서 느린 증가 (안정)
- W_max 초과 시 다시 빠르게 증가 (탐색)

Reno 보다 큰 BDP 에서 압도적.

---

## 8. BBR 자세히

### 8.1 측정

- **BtlBw** (Bottleneck Bandwidth) — 최근 N 패킷의 최대 전송률
- **RTprop** (Round-Trip propagation time) — 최근 N RTT 의 최솟값
- 둘 다 시간 윈도우 (10 sec) 이동 평균

### 8.2 cwnd 계산

```
cwnd = BtlBw × RTprop = BDP
```

→ BDP 만큼만 in-flight. 라우터 큐 안 채움.

### 8.3 Pacing

송신 간격 조절 — 패킷 burst 방지.
```
pacing_rate = BtlBw × gain
```

### 8.4 단계

| 단계 | 동작 |
| --- | --- |
| **STARTUP** | gain=2.89, 지수 탐색 |
| **DRAIN** | gain<1, 큐 비우기 |
| **PROBE_BW** | 8 라운드 gain 변화 (1.25, 0.75, 1, ...) |
| **PROBE_RTT** | 가끔 cwnd 최소화로 RTprop 측정 |

### 8.5 BBR 의 장단

장점:
- **Bufferbloat** 환경 (캐리어 / 모바일) 에 강함
- Latency 낮음

단점:
- CUBIC 등과 공존 시 더 많이 가짐 (BBRv1)
- BBRv2 가 개선
- Loss-based 와 다른 측정 — bug 가능성

---

## 9. ECN — Explicit Congestion Notification

### 9.1 동기
Loss 기다리지 말고 **라우터가 미리 알림**.

### 9.2 동작

```
1. 송신자 SYN 에 ECT (ECN Capable) bit
2. 라우터가 큐 가득 직전 → IP 헤더의 CE (Congestion Experienced) bit set
3. 수신자가 ACK 에 ECE bit
4. 송신자가 cwnd 감소 + CWR bit 응답
```

### 9.3 사용
- 데이터센터 (DCTCP)
- 옛 인터넷에선 미들박스가 ECN bit 무시 / 폐기
- 모던: 점점 활성

### 9.4 L4S (Low Latency Low Loss Scalable Throughput, RFC 9330+)
- ECN 강화 — 1 bit 마킹으로 미세 조정
- Apple iOS / macOS 가 채택 (2022+)

---

## 10. Linux 의 혼잡 제어 선택

```bash
# 현재 알고리즘 보기
sysctl net.ipv4.tcp_congestion_control
# net.ipv4.tcp_congestion_control = cubic

# 사용 가능한 알고리즘
sysctl net.ipv4.tcp_available_congestion_control
# cubic reno bbr

# 변경
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

# 영구
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
```

### 사용 사례
- **CUBIC** — 기본, 안정
- **BBR** — 큰 BDP / bufferbloat 환경 (CDN, 데이터센터)
- **DCTCP** — 데이터센터 ECN 환경

---

## 11. 함정

### 함정 1 — Window 의 분리 무시
rwnd, cwnd 둘 중 작은 것이 실제 윈도우. 어느 쪽이 병목인지 ss 로 확인.

### 함정 2 — IW (Initial Window) 너무 작음
RFC 6928 — IW 10 권장. 짧은 연결에 큰 차이.

### 함정 3 — 같은 알고리즘 양쪽 가정
혼잡 제어는 송신자 측. 수신자 알 필요 X. 비대칭 OK.

### 함정 4 — BBR 의 fairness
BBRv1 은 CUBIC 과 공존 시 더 점유. 같은 환경에서 모두 BBR 권장.

### 함정 5 — ECN 무력화
일부 미들박스가 ECN bit 폐기. `tcp_ecn=2` (수동적) 가 안전.

### 함정 6 — Pacing 없는 BBR
BBR 은 pacing 필수. tc / fq qdisc 와 함께.

```bash
sudo tc qdisc replace dev eth0 root fq
```

---

## 12. 학습 자료

- RFC 5681 (Congestion Control), RFC 6582 (NewReno)
- RFC 8312 (CUBIC), RFC 9002 (QUIC Loss Detection — BBR 영향)
- "Congestion Avoidance and Control" — Van Jacobson 1988
- "BBR: Congestion-Based Congestion Control" — Cardwell et al. 2016
- Linux Kernel `tcp_cubic.c`, `tcp_bbr.c`

---

## 13. 관련

- [[sliding-window]] — Flow Control
- [[tcp-performance-tuning]] — sysctl
- [[../topics/topics]] — Bufferbloat
- [[tcp]] — TCP hub
