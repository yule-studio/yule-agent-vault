---
title: "Session Resumption — Session ID / Ticket / PSK"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T03:10:00+09:00
tags:
  - network
  - tls
  - session-resumption
  - psk
---

# Session Resumption — Session ID / Ticket / PSK

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | TLS 1.2 ID/Ticket vs TLS 1.3 PSK |

**[[tls-ssl|↑ TLS]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

이전 TLS 세션의 비밀을 **재사용** 해 handshake 줄이기. Full → Resumed (1 RTT).
0-RTT 까지.

---

## 2. 동기

### Full handshake 비용
- TLS 1.2 — 2 RTT
- TLS 1.3 — 1 RTT
- 비대칭 암호 (RSA / ECDHE) CPU 비쌈

### Resumption
- 옛 세션의 master secret / PSK 재사용
- Handshake 줄임 + CPU 절약

---

## 3. TLS 1.2 — Session ID

### 동작

```
첫 연결:
Client → ClientHello (Session ID = empty)
Server → ServerHello (Session ID = <random>)
... Full handshake ...

서버: (Session ID, master secret, cipher, ...) 저장

재방문:
Client → ClientHello (Session ID = <previous>)
Server: 저장된 매핑 검색
Server → ServerHello (Session ID = <same>)
Server → ChangeCipherSpec + Finished     ← 즉시 암호화
Client → ChangeCipherSpec + Finished
```

→ 1 RTT — 비대칭 암호 X.

### 단점
- 서버가 세션 저장 — scaling 어려움 (sticky session 또는 공유 store)
- 분산 환경 — Memcached / Redis

---

## 4. TLS 1.2 — Session Ticket (RFC 5077)

### 동작

```
첫 연결 후:
Server → NewSessionTicket
   - 암호화된 ticket (master secret + meta)
Client: 저장

재방문:
Client → ClientHello + ticket
Server: 복호화 → 검증 → resumed
```

→ **서버가 상태 저장 X** — 모든 정보가 ticket 에. Stateless.

### Stateless 장점
- 어떤 서버 인스턴스든 처리
- Sticky session 불필요

### Ticket Key 보안
- 서버가 ticket 암호화 / 복호화에 사용
- 키 노출 → 모든 옛 세션 복호화 가능
- **정기 회전** (24 시간 권장)

```nginx
ssl_session_tickets on;
ssl_session_ticket_key /etc/nginx/ticket.key;     # 회전
```

### 한계 — Forward Secrecy 깨질 위험
- Ticket key 도난 시 — 옛 세션 모두 복호화
- "Forward Secrecy 가 ticket key 의 함수" — 일부 비판
- 해결: 짧은 키 회전 + 인메모리 (Cloudflare 패턴)

---

## 5. TLS 1.3 — PSK (Pre-Shared Key)

TLS 1.2 의 Session ID / Ticket 통합 → PSK.

### 2 종류
- **External PSK** — 사전 설정 (IoT)
- **Resumption PSK** — 이전 세션의 ticket (대부분)

### 동작

```
첫 연결:
... 1 RTT Full handshake ...
Server → NewSessionTicket
  - PSK identity
  - PSK lifetime
  - 0-RTT max_early_data
Client: PSK 저장 (보통 7 일)

재방문:
Client → ClientHello + psk_identity + psk_key_exchange_modes
   key_exchange_modes:
     - psk_ke: pure PSK (RTT 절약, FS X)
     - psk_dhe_ke: PSK + DHE (RTT 절약, FS O)  ← 권장
Server → ServerHello + selected psk
        + 1-RTT data
```

### 0-RTT 확장
```
Client → ClientHello + early_data
                       + Application Data (encrypted with PSK)
```

→ 0 RTT 데이터.

자세히 → [[tls-1-3#6 0-RTT]]

---

## 6. 비교

| 측면 | TLS 1.2 Session ID | TLS 1.2 Ticket | TLS 1.3 PSK |
| --- | --- | --- | --- |
| 서버 상태 | Stateful | Stateless | Stateless |
| Scaling | 어려움 | 쉬움 | 쉬움 |
| Forward Secrecy | DHE 시 ✅ | Ticket key 의존 | psk_dhe_ke 시 ✅ |
| RTT | 1 | 1 | 1 (또는 0) |
| 0-RTT | ❌ | ❌ | ✅ |

---

## 7. Session Resumption 의 보안 고려

### 7.1 Replay Attack (0-RTT)
- 0-RTT 패킷 캡처 → 재전송
- 방어: idempotent only / Anti-replay buffer

### 7.2 Ticket Lifetime
- 짧을수록 안전 — 도난 영향 ↓
- 권장: 1-7 일

### 7.3 Forward Secrecy
- psk_dhe_ke 사용 (TLS 1.3)
- Ticket key 도난 ≠ 옛 세션 복호화 (DHE 의 비밀 키는 일회용)

### 7.4 Pinning
- Cert pinning 과 PSK 의 상호작용
- PSK 가 cert 검증 우회 → 한 번 인증 후 무한 사용 위험
- 정기 cert 검증 강제

---

## 8. 서버 설정

### Nginx
```nginx
# Session ID + Ticket
ssl_session_cache shared:SSL:50m;
ssl_session_timeout 1h;
ssl_session_tickets on;

# Ticket key (회전)
ssl_session_ticket_key /etc/nginx/ticket1.key;
ssl_session_ticket_key /etc/nginx/ticket2.key;    # rolling
```

### HAProxy
```
tune.ssl.cachesize 50000
tune.ssl.lifetime 3600
```

### Cloudflare
- 자체 cluster-wide ticket key
- 정기 회전 (보안)

---

## 9. 클라이언트

### curl
```bash
curl -v --tls-session-cache https://example.com/

# 재방문 — Session ID / Ticket 자동
```

### Python
```python
import ssl
ctx = ssl.create_default_context()
# session 객체로 재사용
```

### Browser
- 모든 모던 브라우저 자동 세션 캐시
- Incognito 는 별도

---

## 10. 도구 확인

```bash
# Session 재사용 확인
openssl s_client -connect example.com:443 -reconnect

# 출력:
# - 첫 연결: Full handshake
# - 재연결: "reused" 표시
# - 만약 "New" 만 → 재사용 X
```

```bash
# Wireshark
display filter: tls.handshake.extensions.psk_identity
                tls.handshake.extension.type == 41 (pre_shared_key)
```

---

## 11. 함정

### 함정 1 — Ticket Key 회전 안 함
도난 시 모든 옛 세션 위험. 24 시간 회전.

### 함정 2 — Session 캐시 크기
큰 사이트 — 메모리 부담. 캐시 크기 조정.

### 함정 3 — 분산 서버의 Ticket key 동기
모든 인스턴스 같은 key — 한 곳만 다르면 resumption X.

### 함정 4 — 0-RTT Replay
공격자가 캡처된 0-RTT 재전송. Idempotent only.

### 함정 5 — psk_ke 만 사용
Forward Secrecy X. psk_dhe_ke 권장.

### 함정 6 — PSK 의 cert 우회
처음 cert 검증 후 PSK 무한 — 짧은 PSK lifetime.

---

## 12. 학습 자료

- **RFC 5077** (TLS Session Ticket)
- **RFC 8446** Section 4.6 (TLS 1.3 PSK)
- **RFC 8470** (Using Early Data — 0-RTT)
- Cloudflare blog — TLS Session Resumption

---

## 13. 관련

- [[tls-ssl]] — TLS hub
- [[tls-handshake-1-2]] — 1.2 Session ID/Ticket
- [[tls-1-3]] — 1.3 PSK / 0-RTT
- [[../http/versions/http-3-quic]] — QUIC 의 0-RTT
