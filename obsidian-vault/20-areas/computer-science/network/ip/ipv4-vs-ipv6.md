---
title: "IPv4 vs IPv6 — 깊은 비교 & Migration"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:20:00+09:00
tags:
  - network
  - ipv4
  - ipv6
  - migration
---

# IPv4 vs IPv6 — 깊은 비교 & Migration

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 비교 / Dual-stack / Happy Eyeballs / Migration 전략 |

**[[ip|↑ IP]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한눈에

| 측면 | IPv4 | IPv6 |
| --- | --- | --- |
| **주소 길이** | 32 bit | 128 bit |
| **주소 공간** | 4.3 × 10⁹ | 3.4 × 10³⁸ |
| **표기** | 192.168.1.1 | 2001:db8::1 |
| **헤더** | 20-60 byte (가변) | 40 byte (고정) |
| **헤더 Checksum** | 있음 | **제거** |
| **단편화** | 송신자 + 라우터 | **송신자만** |
| **최소 MTU** | 576 (1280 권장) | **1280 (강제)** |
| **ARP** | 사용 | **NDP** (ICMPv6) |
| **Multicast** | 옵션 | **표준** |
| **Broadcast** | 있음 | **없음** |
| **자동 구성** | DHCP | **SLAAC** + DHCPv6 |
| **IPsec** | 옵션 | **표준** (실제 선택) |
| **QoS** | DSCP/ECN | DSCP/ECN + **Flow Label** |
| **NAT** | 필수 (사실상) | 보통 X |
| **Mobility** | Mobile IPv4 | Mobile IPv6 |
| **Anycast** | 옵션 | 표준 |
| **End-to-end** | 깨짐 (NAT) | **복원** |

---

## 2. IPv6 의 진짜 동기

### 2.1 표면 — IPv4 주소 부족
- 1990 후반 예상
- 2011 IANA 풀 소진
- RIR 별로 2014-2020 소진

### 2.2 깊은 — End-to-end 복원
- NAT 가 깨뜨린 인터넷의 원래 모델 복원
- P2P 응용 가능
- 서비스 발견 / 라우팅 단순화

### 2.3 단순화
- 라우터 부담 ↓ (단편화 / 체크섬 제거)
- 헤더 고정 40 byte (캐시 친화)
- Extension Headers 로 확장성

---

## 3. 주요 변화 깊이

### 3.1 ARP → NDP

| 측면 | ARP (IPv4) | NDP (IPv6) |
| --- | --- | --- |
| 전송 | Broadcast | Multicast (Solicited-Node) |
| 효율 | 모두에게 | 관심 있는 노드만 |
| 보안 | 약함 (Spoofing) | SEND (옵션, 거의 X) |
| 메시지 | Request/Reply | NS/NA/RS/RA/Redirect |

### 3.2 DHCP → SLAAC + DHCPv6

| | DHCP | SLAAC |
| --- | --- | --- |
| 자동 구성 | 서버 필요 | RA 만 |
| 상태 | Stateful | Stateless |
| DNS 정보 | 포함 | RDNSS option (RFC 8106) |

→ IPv6 는 SLAAC + DHCPv6 (DNS / NTP 등 부가 정보) 결합.

### 3.3 Fragmentation 처리 위치

- IPv4 — 라우터가 단편화 가능
- IPv6 — **라우터는 폐기 + ICMPv6 Type 2** → 송신자가 PMTUD

→ 라우터 CPU 절약, 호스트 책임 증가.

### 3.4 헤더 Checksum 제거

이유:
- L2 CRC (Ethernet 등) 이 보호
- L4 (TCP/UDP) checksum 도
- 라우터 매 hop 재계산 비용 ↓

→ 약간의 신뢰성 trade-off → 매체 / L4 가 책임.

---

## 4. Dual-Stack (IPv4 + IPv6 공존)

```
호스트가 양 stack 모두 활성:
- IPv4 주소 (DHCP 등)
- IPv6 주소 (SLAAC)

응용이 어느 stack 쓸지 선택:
- DNS 조회 → A (IPv4) + AAAA (IPv6)
- 둘 다 있으면? Happy Eyeballs
```

### 단점
- 운영 비용 2 배 (방화벽 / 모니터링 / 라우팅)
- 보안 표면 2 배
- 디버깅 복잡

→ 점진 IPv6-only 로 이동 중.

---

## 5. Happy Eyeballs (RFC 8305)

DNS 가 A + AAAA 모두 응답하면 어느 쪽?

### 알고리즘
1. AAAA (IPv6) 시도 시작
2. 50ms 후 응답 없으면 A (IPv4) 도 동시 시도
3. 먼저 connect() 성공한 쪽 사용

→ IPv6 우선이지만 빠른 fallback. 사용자 경험 ↓ 최소화.

### v2 (RFC 8305)
- Address sorting (preferred prefix 등)
- Cache fresh 결과

---

## 6. Migration 기술

### 6.1 Dual-Stack
- 가장 흔한 — 양 stack 운영
- 두 인프라 모두 필요

### 6.2 Tunneling

#### 6in4 (RFC 4213)
- IPv6 패킷을 IPv4 안에 캡슐화
- Protocol 41
- 옛 broker 서비스 (Hurricane Electric)

#### 6to4 (RFC 3056)
- 자동 IPv6 prefix `2002::/16`
- 옛 — 보안 위험으로 사장

#### Teredo (RFC 4380)
- IPv6 over UDP over IPv4 (NAT 통과)
- Windows 기본 활성 (사장 중)

### 6.3 Translation

#### NAT64 + DNS64
- IPv6-only 클라 ↔ IPv4 서버
- 모바일 통신사 표준

#### 464XLAT (RFC 6877)
- 양쪽 translation
- 안드로이드 모바일 표준

### 6.4 IPv6-only with NAT64

```
Client (IPv6 only) → DNS64 → AAAA 응답 (64:ff9b::8.8.8.8)
Client → 64:ff9b::8.8.8.8 → NAT64 → 8.8.8.8 (IPv4)
```

→ 미국 / 한국 모바일망 (LTE/5G) 대부분 이 구조.

---

## 7. 도입 현황 (2025)

### 트래픽 비율 (Google 측정)
- 글로벌 평균: **45%**
- 미국: 60%+
- 인도: 70%+
- 프랑스 / 독일 / 베트남: 60%+
- 한국: ~30%
- 중국: 30%
- 모바일: 80%+ (이동통신사 대부분)

### 주요 서비스
- AWS, Azure, GCP — 모두 IPv6 지원
- Cloudflare — IPv6 우선
- Facebook, Google — 내부망 IPv6-only
- iOS App Store — IPv6 호환 의무 (2016+)

---

## 8. Migration 단계 가이드

### 단계 1 — 인벤토리
- 어디서 IPv4 의존?
- 응용 / 라이브러리 IPv6 준비?
- 인프라 (방화벽 / LB / 모니터링) IPv6?

### 단계 2 — Dual-Stack 시작
- DNS 에 AAAA 추가
- IPv6 prefix 할당 (ISP / 클라우드)
- 방화벽 IPv6 정책

### 단계 3 — 트래픽 비율 확인
- 모니터링 IPv6 vs IPv4
- 점진 늘림

### 단계 4 — IPv4 의존 제거
- 응용 코드 IP 처리 점검
- IPv6-literal 처리 (URL 의 [::1]:80)
- 로그 / 분석에서 IPv6

### 단계 5 — IPv6-only + NAT64
- IPv4 인프라 제거
- Legacy 는 NAT64 로

---

## 9. 응용 코드의 변화

### 9.1 소켓 API

```c
// IPv4
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_addr.s_addr = inet_addr("192.168.1.1");

// IPv6 (dual-stack)
struct sockaddr_in6 addr6;
addr6.sin6_family = AF_INET6;
inet_pton(AF_INET6, "::1", &addr6.sin6_addr);

// 모던 — getaddrinfo (양 stack)
struct addrinfo hints = {.ai_family = AF_UNSPEC};
getaddrinfo("example.com", "443", &hints, &result);
// result 가 IPv4/IPv6 모두 포함 → 순서대로 시도
```

### 9.2 URL

```
http://192.168.1.1/
http://[2001:db8::1]/        ← IPv6 는 대괄호
http://[2001:db8::1]:8080/   ← 포트와 구분
```

### 9.3 입력 검증

```python
import ipaddress

try:
    ip = ipaddress.ip_address(user_input)
    if ip.is_private: ...
    if ip.version == 4: ...
    if ip.is_link_local: ...
except ValueError:
    # 잘못된 IP
```

---

## 10. 함정

### 함정 1 — IPv4-mapped IPv6 보안 우회
`::ffff:127.0.0.1` 이 localhost 검사 우회 — 입력 검증 시 주의.

### 함정 2 — IPv6 의 ICMP 차단
NDP 안 동작 → 통신 자체 X. IPv4 의 ICMP 와 다르게 절대 차단 X.

### 함정 3 — 응용의 IP 파싱 가정
정규식이 IPv4 만 — IPv6 거부.

### 함정 4 — Log 분석 도구
IPv6 주소 처리 안 함 — 별도 처리.

### 함정 5 — IPv6 의 Privacy Extensions
주소 자주 변함 — IP 기반 차단 / 추적 어려움.

### 함정 6 — Happy Eyeballs 의 50ms 가정
IPv6 가 미세하게 느리면 IPv4 선택 — IPv6 도입 측정 어려움.

### 함정 7 — IPv6 의 NAT 가정
원칙적으로 X. 호스트 직접 노출 → 방화벽 필수.

---

## 11. 학습 자료

- RFC 8200 (IPv6), RFC 8305 (Happy Eyeballs v2), RFC 7755 (SIIT)
- **IPv6 Essentials** (Silvia Hagen)
- **TCP/IP Guide** (Charles Kozierok)
- Google IPv6 통계 — https://www.google.com/intl/en/ipv6/statistics.html
- APNIC IPv6 가이드

---

## 12. 관련

- [[ip]] — IP hub
- [[ipv4-header]], [[ipv6-header]]
- [[cidr-subnetting]]
- [[nat]] — NAT64
- [[../osi-7-layer/layer-3-network/arp]] — NDP
