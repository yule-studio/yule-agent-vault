---
title: "오류 검출 — Parity / Checksum / CRC"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T15:55:00+09:00
tags:
  - network
  - layer-2
  - crc
  - error-detection
  - parity
  - checksum
---

# 오류 검출 — Parity / Checksum / CRC

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Parity / Checksum / CRC 다항식 / FEC 인덱스 |

**[[layer-2-data-link|↑ L2 Data Link]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

> 오류 **검출** (찾기) 와 오류 **정정** (수정) 은 다르다. L2 는 보통 검출만,
> 정정은 L1 의 FEC 또는 L4 의 재전송 (ARQ).

---

## 1. 검출 (Detection) vs 정정 (Correction)

| 기법 | 비용 | 잡음 | 사용 |
| --- | --- | --- | --- |
| **오류 검출** | 작음 (1-4 byte) | 잘못 검출 가능성 작음 | Ethernet, TCP/IP, USB |
| **FEC (정정)** | 큼 (10-50% 오버헤드) | 정정 가능 | 위성, 5G, CD/DVD, QR |
| **ARQ (재전송)** | 왕복 시간 | 100% 신뢰 | TCP, HDLC |

자세히 → [[../layer-1-physical/noise-and-errors#10 오류 정정]]

---

## 2. Parity Bit (패리티 비트)

가장 단순한 검출:

```
Even Parity: 데이터 + parity = 짝수 개 1
Odd Parity:  데이터 + parity = 홀수 개 1

데이터: 1010110  (1 의 개수: 4)
Even Parity: 0   → 합쳐서 10101100 (1 개수 4, 짝수)
Odd Parity:  1   → 합쳐서 10101101 (1 개수 5, 홀수)
```

### 2.1 단점
- **1 비트 오류** 검출 OK
- **2 비트 오류** 검출 불가 (서로 상쇄)
- 정정 불가

### 2.2 사용
- RS-232 시리얼
- ECC RAM (단순 형태)
- 옛 7-bit ASCII + parity = 8 bit

### 2.3 2D Parity (Block Parity)
행 / 열 각각 parity → 1 비트 정정 가능, 2 비트 검출.

---

## 3. Checksum (체크섬)

### 3.1 단순 Sum
모든 byte 더한 후 마지막 byte 만 (overflow 무시).

### 3.2 1's Complement Checksum (Internet Checksum)

TCP / UDP / IP 가 사용 (16 bit):

```python
def checksum(data):
    if len(data) % 2: data += b'\0'
    s = 0
    for i in range(0, len(data), 2):
        s += (data[i] << 8) + data[i+1]
        s = (s & 0xFFFF) + (s >> 16)   # end-around carry
    return ~s & 0xFFFF
```

### 3.3 장단점
- 빠름 (소프트웨어로 쉬움)
- 약함 — 일부 byte swap / 추가 0 못 잡음
- TCP / UDP / IP 는 보조 — Ethernet CRC + L1 BER 가 주
- 그래도 메모리 / 라우터 비트플립 catch

### 3.4 Fletcher / Adler

- **Fletcher checksum** — 단순 sum 보다 강함 (TCP/IP 의 일부)
- **Adler-32** — zlib 사용, CRC 보다 빠르고 약함

---

## 4. CRC (Cyclic Redundancy Check)

### 4.1 기본 아이디어

데이터를 **다항식 (polynomial)** 으로 보고 다른 다항식으로 나눈 **나머지** 를 보냄.
수신자가 같은 나눗셈을 해 나머지가 0 이면 OK.

```
원본 데이터 D (k bit)
생성 다항식 G (n+1 bit)

D || 0^n  ÷  G  →  나머지 R (n bit)

전송:  D || R
수신:  (D || R) ÷ G  →  나머지가 0 이면 OK
```

(`||` 는 연결)

### 4.2 GF(2) 산술 — XOR 기반

CRC 는 GF(2) (Galois Field 2 = 0, 1) 에서 동작:
- 덧셈 = XOR
- 곱셈 = AND
- 뺄셈 = XOR
- 나눗셈 = XOR + shift

### 4.3 다항식 표기

비트 패턴을 다항식으로:

```
1011 = x³ + x¹ + x⁰
10011 = x⁴ + x¹ + x⁰
```

### 4.4 CRC-3 예제

생성 다항식 `1011` (x³ + x¹ + 1):

```
데이터: 1101
오른쪽에 3 개 0 추가: 1101000

  1101000  ÷  1011
- 1011
  ─────
   0110
   01000   ← 4 비트 가져옴
   01011   ← shift, but leading 0 면 next bit
   ...

(GF(2) 긴 나눗셈 — XOR 만 사용)
```

### 4.5 표준 CRC 다항식

| 이름 | 비트 | 다항식 | 사용 |
| --- | --- | --- | --- |
| **CRC-1** (parity) | 1 | x+1 | RS-232 |
| **CRC-8** | 8 | x⁸+x²+x+1 | I²C, ATM |
| **CRC-16-CCITT** | 16 | x¹⁶+x¹²+x⁵+1 | Bluetooth, X.25, HDLC |
| **CRC-16-IBM** | 16 | x¹⁶+x¹⁵+x²+1 | Modbus, USB |
| **CRC-32 (IEEE 802.3)** | 32 | 0x04C11DB7 | **Ethernet, ZIP, PNG** |
| **CRC-32C (Castagnoli)** | 32 | 0x1EDC6F41 | iSCSI, SCTP, ext4 (모던 CPU intrinsic) |
| **CRC-64** | 64 | 0x42F0E1EBA9EA3693 | XZ, REDIS |

### 4.6 CRC-32 (Ethernet FCS)

```
G(x) = x³² + x²⁶ + x²³ + x²² + x¹⁶ + x¹² + x¹¹ + x¹⁰
     + x⁸ + x⁷ + x⁵ + x⁴ + x² + x + 1
```

비트 패턴: `0x04C11DB7`.

### 4.7 CRC 의 강점

- **단일 비트 오류** — 100% 검출
- **2 비트 오류** — 거의 100% (G 가 x+1 인수가 있으면)
- **버스트 오류** (n 비트 이하) — 100% (n = CRC bit 수)
- **임의 오류** — `1 - 2⁻ⁿ` 확률 검출 (CRC-32 면 1 - 2⁻³² ≈ 99.9999999998%)

→ Ethernet 의 BER 10⁻¹² + CRC-32 → 미검출 오류 거의 0.

### 4.8 CRC 하드웨어 구현

```
[Shift Register XOR network]
LFSR (Linear Feedback Shift Register) 로 구현
1 클록당 1 비트 처리 → 매우 빠름
```

x86 의 `crc32` 명령 (SSE4.2) 은 CRC-32C 를 HW 로 — 1 사이클당 byte.

### 4.9 Python 예제

```python
import zlib

data = b"hello world"
crc = zlib.crc32(data)
print(f"CRC-32: {crc:08X}")        # 0D4A1185
```

---

## 5. 헤밍 코드 (Hamming Code) — 정정

오류 정정 코드 (FEC) — 검출 + 정정.

```
Hamming(7,4):  4 비트 데이터 → 3 패리티 → 7 비트 코드
1 비트 오류 정정 / 2 비트 검출
```

ECC RAM, 일부 통신에서 사용.

---

## 6. Reed-Solomon — 버스트 오류 강함

- **블록 코드** — 심볼 (byte) 단위
- 큰 redundancy 로 여러 심볼 정정
- 사용: **CD/DVD, QR Code, DSL, Voyager**

QR Code 의 L/M/Q/H 수준 = Reed-Solomon 의 redundancy 비율.

자세히 → [[../layer-1-physical/noise-and-errors#10 오류 정정]]

---

## 7. LDPC / Turbo / Polar (모던 FEC)

- **LDPC** (Low-Density Parity-Check) — 1962 Gallager, 잊혀졌다가 2000 년 부활
  - 5G, Wi-Fi 6, DVB-S2
- **Turbo Code** — 1993 Berrou — 3G/4G
- **Polar Code** — 2008 Arıkan, Shannon 한계 달성 가능 — 5G 제어

---

## 8. 계층 별 검출 / 정정

```
L1 Physical:  FEC (Reed-Solomon, LDPC) — 정정
L2 Data Link: CRC-32 (Ethernet FCS) — 검출
L3 Network:   IP header checksum (IPv4 만, IPv6 제거) — 검출
L4 Transport: TCP/UDP checksum — 검출, TCP 는 ARQ 정정
L7 Application: HMAC / 디지털 서명 — 무결성 + 인증
```

여러 계층에서 중첩 검출 — defense in depth.

---

## 9. 함정

### 함정 1 — CRC 가 모든 변조를 잡음 가정
악의적 변조는 데이터 + CRC 둘 다 갱신 가능 → **무결성** 은 HMAC 필요.

### 함정 2 — Parity 의 약함 무시
2 비트 오류 못 잡음. RAM 은 ECC (SECDED) 가 표준.

### 함정 3 — TCP Checksum 만 신뢰
16 bit checksum + 약한 알고리즘 — 메모리 오류 / Cosmic ray 미검출 사례 있음.

### 함정 4 — CRC 다항식 선택 부주의
임의 다항식 선택 시 검출률 폭락. 검증된 표준 사용.

### 함정 5 — CRC 의 시작값 / 종료 XOR
구현마다 다름 (CRC-32 in Ethernet vs ZIP). 호환성 확인.

### 함정 6 — Byte order
Big-endian vs Little-endian CRC.

---

## 10. 학습 자료

- Painless Guide to CRC Error Detection Algorithms — Ross Williams (1993)
- **Error Control Coding** (Lin / Costello)
- **Information Theory, Inference, and Learning Algorithms** (MacKay) — 무료 PDF
- Wikipedia CRC 다항식 표
- IEEE 802.3 FCS 정의

---

## 11. 관련

- [[ethernet-frame]] — FCS 위치
- [[../layer-1-physical/noise-and-errors]] — Shannon / FEC
- [[../layer-4-transport/tcp-udp-checksum]] — L4 checksum
- [[layer-2-data-link]] — 상위
