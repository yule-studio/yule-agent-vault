---
title: "L5 — 세션 계층 (Session Layer)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T17:05:00+09:00
tags:
  - network
  - osi
  - layer-5
  - session
---

# L5 — 세션 계층 (Session Layer)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 세션 / Duplex / Checkpoint / RPC / NetBIOS / SOCKS |

**[[../osi-7-layer|↑ OSI 7 계층]]** · **[[../../network|↑↑ network hub]]**

> 현대 TCP/IP 에선 L5/L6 가 응용 (L7) 에 흡수되어 거의 보이지 않는다.
> 이 노트는 학문적 개념 + 일부 살아남은 프로토콜 (RPC, NetBIOS, SOCKS).

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **전송 단위** | data |
| **주요 기능** | 세션 수립 / 유지 / 종료, Duplex 협상, 동기화 / Checkpoint |
| **대표 프로토콜** | RPC, NetBIOS, SAP, SOCKS, NFS (구), SQL session |

---

## 1. 한 줄 정의

**두 응용 사이의 대화 (Dialog) 를 관리** 하는 계층. 연결 수립 / 유지 / 종료,
duplex 협상, 체크포인트, 동기화. 실제 TCP/IP 에선 응용 (L7) 이 흡수.

---

## 2. L5 의 책임

### 2.1 Dialog Control
- 누가 보낼 차례? — Half-duplex 의 토큰
- Full-duplex 면 무관

### 2.2 동기화 (Synchronization)
- 큰 파일 전송 중 중단 → 재개 가능
- Checkpoint = 동기화 지점
- FTP REST, SMB resume

### 2.3 Session 관리
- 시작 / 종료 / 재시작
- 인증
- 인가
- Token / Session ID

---

## 3. RPC — Remote Procedure Call

원격 함수 호출 추상화 — 다른 호스트의 함수를 로컬처럼.

### 3.1 동작

```
Client → Stub (Marshal) → 네트워크 → Stub (Unmarshal) → Server
Server: 함수 실행 → 결과 → 역방향
```

### 3.2 주요 RPC

| 종류 | 설명 |
| --- | --- |
| **Sun RPC / ONC RPC** | NFS, RFC 5531 |
| **DCE / Microsoft DCOM** | Windows |
| **CORBA** | OMG, 사장 |
| **Java RMI** | Java 전용 |
| **XML-RPC / SOAP** | XML, 사장 |
| **JSON-RPC** | 단순 |
| **gRPC** | HTTP/2 + Protobuf — 모던 표준 |
| **Apache Thrift** | Facebook 다언어 |

자세히 → [[../../rpc-messaging/rpc-messaging]]

### 3.3 RPC vs REST

| 측면 | RPC | REST |
| --- | --- | --- |
| 스타일 | 함수 호출 | 자원 조작 |
| 메서드 | 임의 (call) | GET/POST/PUT/DELETE |
| 직렬화 | binary (gRPC) | JSON |
| 성능 | 빠름 | 보통 |
| 발견성 | 낮음 | 높음 (HATEOAS) |

---

## 4. NetBIOS — Network Basic I/O System

### 4.1 개요
- 1983 IBM
- LAN 안 이름 / 데이터그램 / 세션 서비스
- Windows 네트워크의 기반 (옛)
- 137/138/139 포트

### 4.2 NetBIOS Names
- 16-byte 이름 (15 + 1 type)
- LAN 안 broadcast 로 등록 / 해결
- WINS 서버 (Windows Internet Name Service) 가 중앙화

### 4.3 NetBIOS over TCP/IP (NBT)
- Name Service: UDP 137
- Datagram: UDP 138
- Session: TCP 139

### 4.4 SMB (Server Message Block)
- NetBIOS 위에 동작했음
- 현재 SMB 2/3 는 직접 TCP 445
- Samba — Linux 의 SMB 구현

### 4.5 함정
- NetBIOS 가 LAN 안 활성 → 보안 취약점 (WannaCry 가 SMBv1 이용)
- 모던 환경에선 비활성 권장
- IPv6 환경에선 LLMNR 가 대체

---

## 5. SOCKS — Proxy Protocol

### 5.1 개요
- L5 proxy (TCP 또는 UDP)
- HTTP CONNECT 보다 일반적
- SSH dynamic port forwarding (`ssh -D 1080`)

### 5.2 버전

| 버전 | 차이 |
| --- | --- |
| SOCKS4 | TCP, IPv4, 인증 X |
| SOCKS4a | DNS 추가 |
| **SOCKS5** (RFC 1928) | TCP + UDP, IPv6, 다양한 인증, DNS |

### 5.3 SOCKS5 동작

```
1. Client → Proxy: 메서드 협상 (인증 방식)
2. Client → Proxy: 인증 (선택)
3. Client → Proxy: CONNECT 192.0.2.5:443
4. Proxy → 목적지 연결
5. 양방향 데이터 릴레이
```

### 5.4 SOCKS vs HTTP Proxy

| 측면 | SOCKS | HTTP Proxy |
| --- | --- | --- |
| 프로토콜 | 임의 (TCP/UDP) | HTTP / HTTPS |
| 응용 인지 | X | O (URL 수정 가능) |
| 사용 | SSH, Tor, 게임 | 웹 브라우저 |

### 5.5 사용
- **Tor** — Onion routing 의 클라이언트 인터페이스
- **SSH dynamic forwarding** — `-D 1080`
- 회사 방화벽 우회 (정책 의존)

---

## 6. NFS, SMB, AFP — 파일 공유

L5 위에 동작하는 응용 — 세션 관리 포함.

| 프로토콜 | 개발 | 포트 |
| --- | --- | --- |
| **NFSv4** | Sun → IETF | TCP 2049 |
| **SMB 3** | Microsoft | TCP 445 |
| **AFP** | Apple | TCP 548 (사장) |

---

## 7. TLS / SSL 의 L5 측면

엄격히 보면 TLS 는 L5/L6 사이:
- Session 관리 (Session ID, Session Ticket)
- 암호화 / 압축 (L6 적 측면)

실제로는 그냥 "TLS 는 어플리케이션과 TCP 사이 layer" 로 봄.

자세히 → [[../../tls-ssl/tls-ssl]]

---

## 8. 현대 응용에서 L5 의 자리

대부분 응용이 L5/L6 를 직접 구현:
- **WebSocket** — 자체 세션
- **HTTP/2 stream** — 다중 스트림
- **TCP keepalive** — TCP 자체 기능
- **JWT / Cookie** — 세션 표현 (L7)
- **gRPC** — L7 위의 세션

→ "L5 / L6 가 학문적 개념" 인 이유.

---

## 9. 함정

### 함정 1 — OSI 와 TCP/IP 매핑 강요
TCP/IP 는 L5/L6 없음 — 응용에 흡수. 정확한 매핑 어려움.

### 함정 2 — NetBIOS 활성
보안 취약. 비활성.

### 함정 3 — SOCKS 의 인증 없음
SOCKS4 평문, SOCKS5 도 약함. SSH 위 권장.

### 함정 4 — RPC 의 over-engineering
단순 API 에 gRPC 도입 시 과한 경우. REST 충분할 수 있음.

---

## 10. 학습 자료

- RFC 1928 (SOCKS5)
- RFC 5531 (Sun RPC v2)
- **Computer Networks** (Tanenbaum) — L5 부분
- gRPC 공식 문서 — https://grpc.io

---

## 11. 관련

- [[../layer-7-application/layer-7-application]] — 응용
- [[../../rpc-messaging/rpc-messaging]] — RPC 깊이
- [[../osi-7-layer]] — OSI hub
