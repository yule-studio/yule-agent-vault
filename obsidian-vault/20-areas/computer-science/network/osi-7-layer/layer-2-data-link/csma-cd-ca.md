---
title: "CSMA/CD, CSMA/CA, ALOHA — 매체 접근 제어"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T15:50:00+09:00
tags:
  - network
  - layer-2
  - csma
  - mac
  - aloha
---

# CSMA/CD, CSMA/CA, ALOHA — 매체 접근 제어 (Medium Access Control)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | ALOHA / Slotted / CSMA / CD / CA / RTS-CTS / Hidden node |

**[[layer-2-data-link|↑ L2 Data Link]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

---

## 1. 문제 — 누가 매체를 쓸 차례?

**공유 매체 (Shared Medium)** — 한 케이블 / 공기 위에 여러 노드. 동시에 송신하면
**충돌 (Collision)**.

해결법 분류:
1. **Random Access** — 각자 알아서 (ALOHA, CSMA)
2. **Controlled Access** — 순서 / 토큰 (Token Ring, Polling)
3. **Channelization** — 분할 (FDMA, TDMA, CDMA, OFDMA)

---

## 2. ALOHA (1971, 하와이 대학)

### 2.1 Pure ALOHA
- "보내고 싶으면 그냥 보낸다"
- 충돌하면 무작위 시간 대기 후 재전송
- 최대 효율 **18%** (= 1/2e)

### 2.2 Slotted ALOHA
- 시간을 슬롯으로 분할 → 슬롯 시작에만 송신
- 최대 효율 **37%** (= 1/e)

ALOHA 는 단순했지만 효율 낮음 → CSMA 로 발전.

---

## 3. CSMA — Carrier Sense Multiple Access

"**보내기 전에 듣기**" — 다른 송신 있는지 확인.

### 3.1 CSMA 변형

| 변형 | 동작 |
| --- | --- |
| **1-persistent** | 매체 busy → 비기 직후 즉시 송신 (충돌 가능성 ↑) |
| **Non-persistent** | busy → 무작위 시간 대기 후 다시 청취 |
| **p-persistent** | idle → 확률 p 로 송신, (1-p) 로 대기 |

Ethernet 의 CSMA/CD 는 1-persistent.

### 3.2 충돌은 여전히 발생
- 전파 지연으로 둘 다 "idle" 로 들음 → 동시 송신
- 두 메커니즘 필요:
  - **CSMA/CD** — 충돌 감지 (유선)
  - **CSMA/CA** — 충돌 회피 (무선)

---

## 4. CSMA/CD — Collision Detection (옛 이더넷)

```
1. 매체 청취
2. idle 이면 송신
3. 송신 중 충돌 감지 (전압 이상)
4. JAM signal (32 bit) 전송 — 모두에게 알림
5. Binary Exponential Backoff — 무작위 시간 대기
6. 재전송 시도
```

### 4.1 Binary Exponential Backoff
n 번째 충돌 시 0 ~ 2ⁿ - 1 슬롯 중 무작위 선택 (n ≤ 10).
16 번 실패 시 포기.

### 4.2 충돌 도메인의 길이 한계

```
RTT (Round Trip Time) ≤ Frame transmission time
```

10 Mbps + 최소 frame 64 byte = 51.2 μs:
- RTT 51.2 μs → 거리 약 2.5 km (구리), 5 km (광)
- 5-4-3 규칙 — 5 세그먼트 / 4 리피터 / 3 active

### 4.3 왜 CSMA/CD 가 사라졌나

- Switch + Full Duplex 가 보편화 → **충돌 자체가 없음**
- 1000BASE-T 부터 효율 위해 사실상 비활성
- 10GbE 부터 표준에서 제거

→ 현대 wired Ethernet 은 CSMA/CD 안 씀.

---

## 5. CSMA/CA — Collision Avoidance (Wi-Fi)

무선은 충돌 감지 불가 — 송신 중에 자신 신호가 워낙 강해 다른 신호 못 들음.

```
1. 매체 청취 — DIFS (DCF Inter-Frame Space) 만큼
2. idle 유지 → Random Backoff (CW window 안)
3. Backoff 0 도달 → 송신
4. 수신측 ACK (SIFS 후)
5. ACK 없으면 충돌로 간주 — CW × 2 (Exponential Backoff)
6. 재전송
```

### 5.1 IFS (Inter-Frame Space) 종류

| IFS | 시간 (802.11a/g) | 용도 |
| --- | --- | --- |
| **SIFS** (Short) | 16 μs | ACK, RTS/CTS — 최우선 |
| **PIFS** | 25 μs | PCF (drop) |
| **DIFS** | 34 μs | DCF (일반 데이터) |
| **EIFS** (Extended) | 88 μs | 충돌 후 |

작은 IFS 가 더 높은 우선순위.

### 5.2 Contention Window (CW)

- 초기 CW: 15 (802.11a/g)
- 충돌 시 × 2 (15 → 31 → 63 → ... → 1023 → 종료)
- Backoff = random(0, CW) × slot time

### 5.3 NAV (Network Allocation Vector)

- 가상 Carrier Sense
- 다른 노드의 송신 duration 을 헤더에서 읽음 → 그 시간 동안 송신 X

---

## 6. Hidden Node Problem (숨은 노드)

```
A ─── AP ─── B

A 가 B 를 들을 수 없음 (멀어서)
A 와 B 가 동시에 AP 로 송신 → AP 에서 충돌
```

해결: **RTS/CTS** (Request to Send / Clear to Send).

### 6.1 RTS/CTS 흐름

```
A → AP: RTS "보낼게, duration=N"
AP → 모두: CTS "허락, B 는 N 동안 조용히"
A → AP: Data
AP → A: ACK
```

CTS 가 hidden node 까지 들리므로 A 와 B 가 동시 송신 회피.

### 6.2 RTS/CTS 비용
오버헤드 큼 — 큰 프레임에만 사용 (RTS Threshold).

---

## 7. Exposed Node Problem (노출된 노드)

```
A ─── B ─── C ─── D

B 가 A 로 송신 중 → C 가 D 로 송신 가능 (서로 안 들림)
하지만 C 가 B 를 들으니 송신 안 함 → 비효율
```

CSMA 의 한계. 802.11ax (Wi-Fi 6) 의 BSS Color 가 일부 완화.

---

## 8. Token Passing (옛 IBM Token Ring)

- 토큰 (작은 프레임) 이 링을 순환
- 토큰 가진 노드만 송신
- 충돌 X — 효율 보장
- 결점: 토큰 손실 시 복구 복잡
- 사장 (Ethernet 의 단순성 + 가격 승리)

---

## 9. Polling

- Master 가 각 Slave 에게 "보낼 것 있나?" 물음
- Bluetooth, 일부 산업용 (Modbus)
- 충돌 X
- 결점: Master 부담, Latency

---

## 10. TDMA / FDMA / CDMA / OFDMA

| 방식 | 분할 | 사용 |
| --- | --- | --- |
| **FDMA** | 주파수 | AM/FM 라디오, 1G |
| **TDMA** | 시간 슬롯 | 2G (GSM), Bluetooth |
| **CDMA** | 코드 | 3G (CDMA2000, WCDMA) |
| **OFDMA** | 직교 부반송파 | 4G/5G, Wi-Fi 6 |

OFDMA 는 한 채널을 여러 사용자에 동시 분할 → 효율 ↑.

---

## 11. Wi-Fi 의 매체 접근 진화

| 표준 | 방식 |
| --- | --- |
| 802.11 (1997) | CSMA/CA (DCF) |
| 802.11a/b/g | CSMA/CA + RTS/CTS |
| 802.11n (Wi-Fi 4) | + Block ACK |
| 802.11ac (Wi-Fi 5) | + MU-MIMO (down) |
| **802.11ax (Wi-Fi 6)** | **OFDMA + MU-MIMO (up/down) + BSS Coloring** |
| **802.11be (Wi-Fi 7)** | **Multi-Link Operation (MLO)** — 동시 여러 대역 |

---

## 12. 면접 / 토픽

1. **CSMA/CD 동작** — Carrier sense, Collision detect, JAM, Backoff.
2. **CSMA/CD 가 왜 사라졌나** — Switch + Full Duplex.
3. **CSMA/CA 와 CSMA/CD 차이** — 충돌 회피 vs 감지.
4. **RTS/CTS 의 목적** — Hidden node.
5. **ALOHA 의 효율** — Pure 18% / Slotted 37%.
6. **Binary Exponential Backoff**.
7. **OFDMA 가 Wi-Fi 6 에 가져온 변화**.

---

## 13. 함정

### 함정 1 — Switched Ethernet 에서 CSMA/CD 활성 가정
Full Duplex 면 비활성. 통계에 "Late Collision" 보이면 duplex mismatch.

### 함정 2 — Wi-Fi 의 "충돌 회피" = 충돌 없음 가정
회피일 뿐 완전히 막진 못함. Hidden node 시 자주 발생.

### 함정 3 — RTS/CTS 항상 켜기
짧은 프레임엔 오히려 오버헤드. RTS Threshold 조정.

### 함정 4 — 2.4 GHz 채널 중첩
1/6/11 만 비중첩. 인접 채널 동시 사용 시 모두 충돌처럼 동작.

### 함정 5 — Wi-Fi 사용자 많을 때
DCF 의 Contention Window 가 폭증 → 모두 느려짐. OFDMA 로 완화.

---

## 14. 학습 자료

- IEEE 802.11 표준 — DCF/PCF
- **Computer Networks** (Tanenbaum) — Ch. 4
- **CWNA Study Guide** — Wi-Fi 인증
- Stallings — Wireless Communications

---

## 15. 관련

- [[ethernet-frame]] — Ethernet 프레임 (옛 CSMA/CD)
- [[wireless-link]] — Wi-Fi MAC 깊이
- [[../layer-1-physical/transmission-media]] — 매체별 MAC 결정
- [[layer-2-data-link]] — 상위
