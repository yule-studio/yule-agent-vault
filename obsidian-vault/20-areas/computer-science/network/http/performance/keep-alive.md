---
title: "HTTP Keep-Alive — Persistent Connection"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:40:00+09:00
tags:
  - network
  - http
  - performance
  - keep-alive
---

# HTTP Keep-Alive — Persistent Connection

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Keep-Alive / HTTP/2 multiplexing 와 비교 |

**[[performance|↑ Performance]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

한 TCP 연결로 **여러 HTTP 요청** 처리. HTTP/1.1 부터 기본. 매 요청 새 TCP 핸드셰이크 비용 절약.

---

## 2. 동기

### 옛 HTTP/1.0 — 매 요청 새 TCP

```
1 페이지 = HTML + image × 10 + CSS × 2 + JS × 3 = 16 요청
→ 16 × (TCP handshake + slow start)
→ 큰 latency
```

### Keep-Alive — 재사용

```
TCP 1 개 연결
├ Request 1
├ Response 1
├ Request 2
├ Response 2
...
└ Request 16
```

---

## 3. 헤더

### HTTP/1.1 — 기본 활성

```http
GET / HTTP/1.1
Host: example.com
(Connection: keep-alive 명시 안 해도 기본)
```

### HTTP/1.0 — 명시

```http
GET / HTTP/1.0
Connection: keep-alive
```

### 종료 — Connection: close

```http
Connection: close
```

→ 이 응답 후 종료.

### Keep-Alive 파라미터

```http
Keep-Alive: timeout=5, max=100
```

- `timeout` — idle timeout (초)
- `max` — 최대 요청 수

---

## 4. 흐름

```
1. TCP handshake (1 RTT)
2. TLS handshake (1-2 RTT, HTTPS 시)
3. Request 1 → Response 1
4. Request 2 → Response 2  ← 같은 TCP
...
5. idle timeout 또는 max 도달 시 종료
6. Close 또는 새 연결
```

---

## 5. 한계 — HTTP/1.1 Pipelining 의 실패

HTTP/1.1 의 pipelining (응답 안 기다리고 여러 요청) 은 사실상 사장:
- HoL Blocking — 첫 응답 느리면 모든 응답 stuck
- 미들박스 호환성 X
- 브라우저 기본 비활성

→ 결과: **한 TCP 연결에 순차적 요청만**.

자세히 → [[pipelining]]

---

## 6. 브라우저의 연결 수 한계

```
한 도메인 당 6 개 TCP 연결 (Chrome / Firefox)
→ 7 번째 요청은 기존 연결 대기
```

### 옛 우회 — Domain Sharding

```
img1.example.com
img2.example.com
img3.example.com
→ 도메인 별 6 연결 → 18 연결
```

→ HTTP/2 의 멀티플렉싱이 표준화 후 안티패턴.

---

## 7. HTTP/2 의 진화 — Multiplexing

```
HTTP/1.1: 1 TCP = 순차 요청 (또는 6 개 병렬 TCP)
HTTP/2:   1 TCP = 수십~수천 stream (병렬)
HTTP/3:   1 QUIC = 수십~수천 stream + HoL 해결
```

자세히 → [[../versions/http-2]]

---

## 8. Keep-Alive vs Multiplexing

| 측면 | Keep-Alive (1.1) | Multiplexing (2/3) |
| --- | --- | --- |
| 연결 | 1 TCP / 6 병렬 | 1 TCP / QUIC |
| 동시 요청 | 순차 (1 TCP 안) | 수많은 stream |
| HoL Blocking | 응용 + TCP 레벨 | 응용 해결, TCP 잔존 (HTTP/2) / 완전 해결 (HTTP/3) |
| 헤더 압축 | X | HPACK / QPACK |

---

## 9. 서버 설정

### Nginx
```nginx
keepalive_timeout 65;
keepalive_requests 1000;

# Upstream (백엔드와의 keep-alive)
upstream backend {
    server 10.0.0.1:8080;
    keepalive 32;        # 풀 크기
    keepalive_requests 10000;
    keepalive_timeout 60s;
}
```

### Apache
```apache
KeepAlive On
KeepAliveTimeout 5
MaxKeepAliveRequests 100
```

### Express
```javascript
const server = app.listen(...);
server.keepAliveTimeout = 65000;       // 5s default
server.headersTimeout = 66000;         // > keepAliveTimeout
```

---

## 10. 클라 (라이브러리)

### curl
```bash
curl -v https://example.com/path1 https://example.com/path2
# → 같은 TCP 재사용
```

### Python requests
```python
import requests
session = requests.Session()
session.get('https://example.com/path1')
session.get('https://example.com/path2')   # 같은 TCP
```

### Node.js
```javascript
const https = require('https');
const agent = new https.Agent({keepAlive: true, maxSockets: 50});
fetch('https://example.com/', {agent});
```

### Java HttpClient
```java
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(10))
    .build();
// 자동 connection pooling
```

---

## 11. Connection Pool

응용 / proxy / LB 의 패턴:

```
앱 ↔ Pool ↔ Backend
       ↑
       N 개 connection 풀
       재사용
```

### 효과
- TCP / TLS handshake 비용 ↓
- 즉시 사용 가능
- 더 큰 throughput

### 파라미터
- max connections (총)
- max idle (대기 중)
- idle timeout
- max age (연결 수명)

---

## 12. 함정

### 함정 1 — keepAliveTimeout < LB timeout
```
LB: 60s 후 close
서버: 10s idle 후 close
→ LB 가 dead connection 사용 → 502 에러
```

해결: LB > 서버.

### 함정 2 — Connection pool 의 크기
- 너무 작음 → 연결 부족 → 새로 만듬 (느림)
- 너무 큼 → 메모리 + 백엔드 부하

### 함정 3 — 옛 HTTP/1.0 호환
명시 `Connection: keep-alive` 보내야.

### 함정 4 — Domain Sharding 의 안티패턴
HTTP/2 에선 오히려 비효율.

### 함정 5 — TIME_WAIT 누적
짧은 연결 폭주 시 — Keep-Alive 가 해결. 자세히 → [[../../tcp/four-way-termination]]

### 함정 6 — Idle Connection 의 누수
사용자 응용이 close 안 함 — keepalive 가 영원히 유지 (timeout 까지).

---

## 13. 학습 자료

- **RFC 9112** Section 9.3 (Connection Management)
- "High Performance Browser Networking" Ch. 11
- "HTTP keep-alive" — Nginx blog

---

## 14. 관련

- [[performance]] — Performance hub
- [[pipelining]] — 실패한 시도
- [[../versions/http-2]] — Multiplexing
- [[../../tcp/four-way-termination]] — TIME_WAIT
