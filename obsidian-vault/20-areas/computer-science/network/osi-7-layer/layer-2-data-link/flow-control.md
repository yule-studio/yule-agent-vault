---
title: "흐름 제어 (Flow Control)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T16:00:00+09:00
tags:
  - network
  - layer-2
  - flow-control
  - sliding-window
---

# 흐름 제어 (Flow Control)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Stop&Wait / Sliding Window / HDLC / 802.3x PAUSE / PFC |

**[[layer-2-data-link|↑ L2 Data Link]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

> L2 의 흐름 제어. L4 TCP 의 윈도우 제어와 개념은 같지만 다른 계층.

---

## 1. 왜 흐름 제어가 필요한가

- 송신자 빠름 + 수신자 느림 → 버퍼 overflow → 패킷 손실
- 빠른 송신을 적당히 늦추기 — **수신자 보호**

vs 혼잡 제어 (Congestion Control):
- 흐름 제어 = 수신자 보호
- 혼잡 제어 = 네트워크 보호

---

## 2. Stop-and-Wait (정지-대기) 프로토콜

```
Sender: Send F1 → wait
Receiver: F1 → ACK1
Sender: Receive ACK1 → Send F2 → wait
...
```

### 2.1 동작
한 프레임 보내고 ACK 받기 전까지 다음 프레임 X.

### 2.2 효율
```
Efficiency = T_frame / (T_frame + 2 × T_propagation)
```

위성 (long RTT) 에서 매우 비효율:
- T_frame = 1 ms, RTT = 500 ms → 효율 < 1%

### 2.3 단순함의 장점
- 구현 쉬움
- 정확함
- 짧은 link 에선 OK (UART, Serial)

---

## 3. Sliding Window (슬라이딩 윈도우)

여러 프레임 한꺼번에 보낸 후 ACK 들 받기 — **파이프라이닝**.

### 3.1 동작

```
Window size = 4

Sender:    F1 F2 F3 F4 ... wait
Receiver:  ACK1 ACK2 ACK3 ACK4
Sender:    F5 F6 F7 F8 (window 슬라이드)
```

### 3.2 윈도우 크기 결정

```
Optimal W = (Bandwidth × RTT) / Frame size
```

1 Gbps + RTT 1 ms + 1500 byte frame:
- W = 10⁹ × 10⁻³ / (1500 × 8) ≈ 83 프레임

### 3.3 윈도우 표현 비트

Sequence number n bit 면:
- Go-Back-N: 윈도우 ≤ 2ⁿ - 1
- Selective Repeat: 윈도우 ≤ 2ⁿ⁻¹

---

## 4. ARQ — 재전송 방식

### 4.1 Stop-and-Wait ARQ
- 한 프레임씩
- ACK 못 받으면 timeout 후 재전송

### 4.2 Go-Back-N ARQ
- 윈도우 N 만큼 보냄
- 손실 발견 시 그 프레임 + 이후 모두 재전송
- 단순, 효율은 낮을 수 있음

### 4.3 Selective Repeat (SR) ARQ
- 손실 프레임만 재전송
- 효율 ↑, 복잡 ↑
- 수신측 버퍼 + 순서 재배열 필요

### 4.4 비교

| 기준 | Go-Back-N | Selective Repeat |
| --- | --- | --- |
| 재전송 | 손실 이후 모두 | 손실만 |
| 수신 버퍼 | 1 프레임 | 윈도우 크기 |
| 복잡도 | 낮음 | 높음 |
| 효율 (BER 높을 때) | 낮음 | 높음 |
| 예 | HDLC | TCP SACK |

---

## 5. HDLC (High-level Data Link Control)

IBM SDLC → ISO 표준. PPP / 시리얼 / WAN 의 기반.

### 5.1 프레임 구조

```
┌────────┬────────┬─────────┬──────────┬──────┬────────┐
│ Flag   │Address │ Control │  Data    │ FCS  │ Flag   │
│  7E    │  1-2   │  1-2    │   var    │  2-4 │  7E    │
└────────┴────────┴─────────┴──────────┴──────┴────────┘
```

### 5.2 프레임 종류 (Control 필드)

| 종류 | 약자 | 용도 |
| --- | --- | --- |
| **I-frame** | Information | 데이터 + 흐름 / 오류 제어 |
| **S-frame** | Supervisory | ACK / NAK / RR / RNR |
| **U-frame** | Unnumbered | 제어 (연결 설정 / 해제) |

### 5.3 S-frame 종류

- **RR** (Receive Ready) — 다음 프레임 받을 준비
- **RNR** (Receive Not Ready) — busy
- **REJ** (Reject) — Go-Back-N
- **SREJ** (Selective Reject) — Selective Repeat

### 5.4 Window 크기
- 3-bit seq → window 7
- 7-bit seq (extended) → window 127

---

## 6. PPP (Point-to-Point Protocol)

HDLC 기반 + 확장 — 다이얼업 / DSL / PPPoE.

```
┌────────┬────────┬─────────┬──────────┬──────────┬──────┬────────┐
│ Flag   │Address │ Control │ Protocol │  Data    │ FCS  │ Flag   │
│  7E    │  FF    │  03     │   2 byte │   var    │  2-4 │  7E    │
└────────┴────────┴─────────┴──────────┴──────────┴──────┴────────┘
```

### 6.1 단계

1. **LCP** (Link Control Protocol) — 링크 협상 (인증, 압축, MTU)
2. **인증** (선택) — PAP, CHAP, EAP
3. **NCP** (Network Control Protocol) — L3 프로토콜 별 (IPCP, IPv6CP)
4. **데이터 송수신**
5. **종료** — LCP terminate

### 6.2 PPPoE (PPP over Ethernet)

- DSL 모뎀이 사용
- Ethernet 위에 PPP 프레임 — 8 byte 헤더 → MTU 1492

---

## 7. 802.3x PAUSE Frame (Ethernet Flow Control)

### 7.1 동작

```
수신측 버퍼 가득 → PAUSE frame 송신
PAUSE: "X 시간 동안 안 보내" (X = quanta)

Quantum = 512 bit times
10 Gbps 에서 1 quantum = 51.2 ns

PAUSE Type = 0x8808, opcode 0x0001
```

### 7.2 한계
- 전체 트래픽 일시 중단 — 우선순위 구분 X
- 한 트래픽 흐름 때문에 모든 트래픽 멈춤

### 7.3 PFC (Priority Flow Control, 802.1Qbb)

- VLAN 의 PCP (Priority Code Point) 별로 PAUSE
- 8 우선순위 각각 독립 흐름 제어
- DCB (Data Center Bridging) 의 일부
- iSCSI / FCoE / RoCE 가 필요로 함

---

## 8. Lossless Ethernet — DCB (Data Center Bridging)

데이터센터에서 "lossless" 가 필요한 경우:
- iSCSI / FCoE — Storage
- RoCE (RDMA over Converged Ethernet) — Low-latency

### 8.1 DCB 구성

- **PFC** (802.1Qbb) — 우선순위별 흐름 제어
- **ETS** (802.1Qaz) — 대역폭 할당
- **DCBX** (802.1Qaz) — 자동 협상
- **QCN** (802.1Qau) — 혼잡 알림

### 8.2 ECN (Explicit Congestion Notification)

- L3/L4 의 ECN — 라우터가 IP 헤더 bit 설정
- TCP / DCQCN (DC) 이 반응

---

## 9. 흐름 제어 vs 혼잡 제어 vs QoS

| 개념 | 보호 대상 | 계층 |
| --- | --- | --- |
| **흐름 제어 (Flow)** | 수신자 버퍼 | L2 / L4 |
| **혼잡 제어 (Congestion)** | 네트워크 전체 | L4 (TCP) |
| **QoS** | 트래픽 우선순위 | L2/L3 |

자세히:
- [[../layer-4-transport/tcp]] — TCP 윈도우 / 혼잡 제어
- [[vlan-stp-bonding]] — PCP / QoS 우선순위

---

## 10. TCP 윈도우 vs L2 흐름 제어

| 측면 | L2 (802.3x) | L4 TCP |
| --- | --- | --- |
| 범위 | 인접 노드 | 종단-종단 |
| 단위 | 시간 (quanta) | byte (window size) |
| 정밀도 | 거친 | 세밀 |
| 우선순위 | PFC 만 | 없음 |
| 보편성 | 데이터센터 / 직접 | 모든 인터넷 |

---

## 11. Buffer Bloat

- 라우터 / NIC 의 큰 버퍼 → ACK 지연 → TCP 가 느리게 반응
- 처리량은 같지만 latency 급증
- 해결: AQM (Active Queue Management) — CoDel, FQ-CoDel, BBR

---

## 12. 함정

### 함정 1 — Stop-and-Wait 의 비효율 무시
긴 link 에서 ACK 대기 시간 ↑ → 처리량 ↓.

### 함정 2 — 802.3x PAUSE 의 부작용
한 NIC 의 buffer full → 전체 stop. Head-of-line blocking.

### 함정 3 — PFC 의 데드락 (PFC deadlock)
순환 PAUSE 로 영구 정지. 그래프 구조 설계 필요.

### 함정 4 — TCP 윈도우와 L2 PAUSE 의 상호작용
잘못 설정 시 throughput 폭락.

### 함정 5 — Jumbo Frame + Buffer 부족
큰 프레임 → 빈 버퍼 적음 → PAUSE 빈번.

### 함정 6 — Wi-Fi 의 흐름 제어 약함
802.11 의 ACK + Block ACK 가 주. 무선 자체 한계로 throughput 변동 큼.

---

## 13. 학습 자료

- **Computer Networks** (Tanenbaum) — Ch. 3.4-3.6
- IEEE 802.3x / 802.1Qbb 표준
- **Data Center Networks** (Liang Wang)
- BufferBloat — bufferbloat.net

---

## 14. 관련

- [[../layer-4-transport/tcp]] — TCP 윈도우
- [[vlan-stp-bonding]] — QoS / PCP
- [[error-detection-crc]] — 검출 후 재전송 ARQ
- [[layer-2-data-link]] — 상위
