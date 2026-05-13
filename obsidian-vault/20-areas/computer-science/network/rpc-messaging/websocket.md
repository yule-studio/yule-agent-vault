---
title: "WebSocket — 양방향 통신"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T08:15:00+09:00
tags:
  - network
  - websocket
  - real-time
---

# WebSocket — 양방향 통신

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | RFC 6455 / Handshake / Frame |

**[[rpc-messaging|↑ RPC/Messaging]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **표준** | RFC 6455 (2011) |
| **포트** | 80 (ws://), 443 (wss://) |
| **시작** | HTTP Upgrade |
| **모델** | 양방향 / persistent |

---

## 1. 한 줄 정의

**HTTP Upgrade 후 TCP 위에 양방향 메시지** — 채팅 / 게임 / 실시간 알림.

---

## 2. HTTP 와의 차이

### HTTP
```
요청 → 응답 → 끊김
```

### WebSocket
```
요청 (Upgrade) → 응답 (101 Switching Protocols)
↓
양방향 메시지 (오랫동안)
```

---

## 3. 핸드셰이크

### Request
```http
GET /ws HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: chat, superchat
Origin: https://example.com
```

### Response
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

### Sec-WebSocket-Accept
```
base64(SHA1(key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))
```

→ Server 가 spec 따르는지 검증.

---

## 4. Frame 포맷

```
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len | Extended payload length       |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |                               |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               | Masking-key, if MASK set to 1 |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

### Opcode
| | 의미 |
| --- | --- |
| 0x0 | continuation |
| 0x1 | text |
| 0x2 | binary |
| 0x8 | close |
| 0x9 | ping |
| 0xA | pong |

### Mask
- Client → Server — masking 필수 (보안)
- Server → Client — masking 안 함

---

## 5. JavaScript API

### Client
```javascript
const ws = new WebSocket('wss://example.com/ws');

ws.onopen = () => {
    console.log('Connected');
    ws.send('Hello server!');
};

ws.onmessage = (event) => {
    console.log('Received:', event.data);
};

ws.onclose = () => console.log('Disconnected');
ws.onerror = (err) => console.error(err);

// 종료
ws.close(1000, 'Goodbye');
```

### Sub-protocol
```javascript
const ws = new WebSocket('wss://example.com/ws', ['chat', 'superchat']);
```

---

## 6. 서버 — Node.js (ws)

```javascript
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws, req) => {
    ws.on('message', (data) => {
        console.log('Received:', data.toString());
        ws.send(`Echo: ${data}`);
    });
    
    ws.on('close', () => console.log('Disconnected'));
    
    ws.send('Welcome!');
});

// Broadcast
function broadcast(data) {
    wss.clients.forEach((client) => {
        if (client.readyState === WebSocket.OPEN) {
            client.send(data);
        }
    });
}
```

---

## 7. 서버 — Go (gorilla/websocket)

```go
import "github.com/gorilla/websocket"

var upgrader = websocket.Upgrader{}

func wsHandler(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil { return }
    defer conn.Close()
    
    for {
        msgType, msg, err := conn.ReadMessage()
        if err != nil { break }
        conn.WriteMessage(msgType, msg)
    }
}
```

---

## 8. Close codes

| Code | 의미 |
| --- | --- |
| 1000 | Normal closure |
| 1001 | Going away |
| 1002 | Protocol error |
| 1003 | Unsupported data |
| 1006 | Abnormal (no close frame) |
| 1008 | Policy violation |
| 1009 | Message too big |
| 1011 | Server error |
| 4000-4999 | Application |

---

## 9. Ping / Pong — Heartbeat

### 목적
- Connection 살아있는지
- NAT timeout 방지

### 자동 (브라우저)
- 브라우저 — 자동 (API 노출 X)
- Server-initiated

### Server (ws)
```javascript
const interval = setInterval(() => {
    wss.clients.forEach((ws) => {
        if (!ws.isAlive) return ws.terminate();
        ws.isAlive = false;
        ws.ping();
    });
}, 30000);

ws.on('pong', () => ws.isAlive = true);
```

---

## 10. WebSocket vs SSE vs Long Polling

| | WebSocket | SSE | Long Polling |
| --- | --- | --- | --- |
| **방향** | 양방향 | 서버 → 클라 | 클라 → 서버 |
| **포맷** | text / binary | text only | HTTP |
| **재연결** | 수동 | 자동 (Last-Event-ID) | 자동 |
| **방화벽** | wss (443) — OK | https (443) — OK | https — OK |
| **HTTP cache / proxy** | bypass | OK | OK |
| **부하** | 적음 | 적음 | 보통 |

자세히 → [[sse]]

---

## 11. 라이브러리 / 추상화

### Socket.IO
- WebSocket + 폴백 (long polling)
- Room / namespace
- 자동 재연결
- Acknowledgment

```javascript
io.on('connection', (socket) => {
    socket.join('room1');
    socket.on('msg', (data, cb) => {
        io.to('room1').emit('msg', data);
        cb({ status: 'ok' });
    });
});
```

### SignalR (Microsoft)
- ASP.NET — WebSocket + 폴백
- .NET / JS 클라

### Phoenix Channels (Elixir)
- WebSocket + presence
- 매우 확장성

### Centrifugo
- 독립 server
- pub-sub / WebSocket / SSE

---

## 12. Scaling

### 문제
- WebSocket — long-lived → connection pin
- Backend scale-out 시 — broadcast 어려움

### 해결
- **Redis pub-sub** — 모든 backend 가 broadcast
- **Sticky session** — 한 사용자 한 backend
- **Stateful edge** — Cloudflare Durable Objects

### 한계
- 10k connection per server (typical)
- C10K → C10M → Phoenix (2M+)

자세히 → [[../topics/c10k-c10m]] (TBD)

---

## 13. 보안

### Origin check
- Server — Origin 헤더 검증
- CSWSH (Cross-Site WebSocket Hijacking) 방어

### Authentication
- 핸드셰이크 시 cookie / token
- 또는 첫 message 로 auth

### TLS
- wss:// (TLS) 필수
- 평문 ws:// — MITM

### Rate limiting
- per IP / per user
- Message rate

---

## 14. 함정

### 함정 1 — Proxy / LB 의 WebSocket 지원
옛 — Upgrade 헤더 처리 안 함. Nginx — proxy_set_header 명시.

### 함정 2 — Idle timeout
60s 같은 timeout — 끊김. Ping / pong 30s.

### 함정 3 — Browser tab background
브라우저 — background 탭의 WebSocket 제한 / 정지.

### 함정 4 — Reconnect storm
서버 재시작 → 모든 클라가 동시 재연결 → 폭주.
Exponential backoff + jitter.

### 함정 5 — Memory leak
Connection 별 listener / state — 누수.

### 함정 6 — Message size
큰 message — RFC 권장 X. Chunking / pagination.

### 함정 7 — WebSocket 의 HTTP cache
경유 시 — cache X (Upgrade). OK.

---

## 15. WebTransport — HTTP/3 의 후속

### 정의
- HTTP/3 (QUIC) 위의 양방향
- 신뢰 / 비신뢰 stream 둘 다

### 특징
- WebSocket 의 모던 대체
- 게임 / VR / 실시간 (UDP 위)

### 표준
- W3C draft / IETF

자세히 → [[../quic/http-3]]

---

## 16. 학습 자료

- **RFC 6455** (WebSocket)
- "High Performance Browser Networking" (Grigorik) — WebSocket
- "Socket.IO" docs
- "Phoenix Channels" guides

---

## 17. 관련

- [[rpc-messaging]] — Hub
- [[sse]] — 단방향 비교
- [[../http/versions/http-2]] — HTTP/2 의 WebSocket (RFC 8441)
- [[webrtc]] — P2P 비교
