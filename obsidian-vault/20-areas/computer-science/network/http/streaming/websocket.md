---
title: "WebSocket"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T01:20:00+09:00
tags:
  - network
  - http
  - streaming
  - websocket
---

# WebSocket

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | RFC 6455 / Upgrade / Frame / 사용 |

**[[streaming|↑ Streaming]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

브라우저 ↔ 서버 **양방향 full-duplex** TCP 통신. HTTP Upgrade 로 시작 후 자체 프로토콜.
RFC 6455 (2011).

---

## 2. 동기

### HTTP 의 한계
- 단방향 — 서버가 클라에게 능동 송신 X
- Polling / Long polling — 비효율
- SSE — 단방향만

### WebSocket
- **양방향** — 서버가 즉시 push
- **저지연** — 한 TCP 연결 재사용
- **binary** — 효율적

---

## 3. Handshake — HTTP Upgrade

### Client → Server
```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://example.com
```

### Server → Client
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### Sec-WebSocket-Accept 계산
```python
import base64, hashlib
KEY = "dGhlIHNhbXBsZSBub25jZQ=="
GUID = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
sha1 = hashlib.sha1((KEY + GUID).encode()).digest()
accept = base64.b64encode(sha1).decode()
# → s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

→ 브라우저가 검증 → handshake 성공.

---

## 4. 이후 — Frame

```
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |                               |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
:                                                               :
+ - - - - - - - - - - - - - - - + - - - - - - - - - - - - - - - +
|    Masking-key (32, client→server 만)         |
+-------------------------------+-------------------------------+
|                          Payload Data                          |
+---------------------------------------------------------------+
```

### Opcode
- 0x0 — Continuation
- 0x1 — Text (UTF-8)
- 0x2 — Binary
- 0x8 — Close
- 0x9 — Ping
- 0xA — Pong

### Masking
- Client → Server: 모든 frame masked (XOR 키)
- Server → Client: masked X
- 보안 (Proxy cache 의 confused deputy 공격 방어)

---

## 5. Browser API

```javascript
const ws = new WebSocket('wss://example.com/chat');

ws.onopen = () => {
    console.log('Connected');
    ws.send('Hello!');
    ws.send(JSON.stringify({type: 'message', text: 'Hi'}));
};

ws.onmessage = (event) => {
    console.log('Received:', event.data);
};

ws.onerror = (err) => {
    console.error('Error:', err);
};

ws.onclose = (event) => {
    console.log('Closed:', event.code, event.reason);
};

// 종료
ws.close(1000, 'Normal close');
```

### Binary
```javascript
ws.binaryType = 'arraybuffer';   // 또는 'blob'
ws.send(new Uint8Array([1, 2, 3]).buffer);
```

---

## 6. 서버 구현

### Node.js — ws 라이브러리
```javascript
const WebSocket = require('ws');
const wss = new WebSocket.Server({port: 8080});

wss.on('connection', (ws, req) => {
    ws.on('message', (data) => {
        // Broadcast
        wss.clients.forEach(client => {
            if (client.readyState === WebSocket.OPEN) {
                client.send(data.toString());
            }
        });
    });
});
```

### Python — FastAPI
```python
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def ws(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        await websocket.send_text(f"Echo: {data}")
```

### Go — gorilla/websocket
```go
var upgrader = websocket.Upgrader{}

func handler(w http.ResponseWriter, r *http.Request) {
    conn, _ := upgrader.Upgrade(w, r, nil)
    for {
        mt, msg, err := conn.ReadMessage()
        if err != nil { break }
        conn.WriteMessage(mt, msg)
    }
}
```

---

## 7. URL Scheme

```
ws://example.com/path        ← HTTP
wss://example.com/path       ← HTTPS (권장)
```

→ HTTPS 환경은 wss 만.

---

## 8. Ping / Pong (Keep-alive)

```
서버: ping (opcode 0x9)
클라: pong (opcode 0xA)
```

### 자동 (라이브러리)
- 보통 30 초 - 1 분 마다
- timeout 시 연결 종료

### 응용 레벨
- 일부 응용은 텍스트 메시지로 자체 ping (`{"type":"ping"}`)
- Discord / Slack 등 — 30 초 마다

---

## 9. Close Code

```
1000 - Normal Closure
1001 - Going Away (페이지 이동)
1002 - Protocol Error
1003 - Unsupported Data
1006 - Abnormal Closure (네트워크)
1008 - Policy Violation
1011 - Server Error
3000-3999 - 응용 정의
4000-4999 - 사설
```

---

## 10. 사용 사례

### 10.1 Chat
- Slack, Discord, WhatsApp
- 실시간 메시지

### 10.2 게임
- 멀티플레이어 상태 동기
- 60-120 Hz 입력 / 상태

### 10.3 라이브 협업
- Google Docs, Figma, Notion
- 동시 편집 (CRDT + WebSocket)

### 10.4 주식 시세
- 실시간 차트
- HFT

### 10.5 알림
- 모바일 / 웹 push

---

## 11. WebSocket vs HTTP/2 vs HTTP/3 vs SSE

| 측면 | WebSocket | HTTP/2 | HTTP/3 | SSE |
| --- | --- | --- | --- | --- |
| 양방향 | ✅ | ✅ (stream) | ✅ | ❌ |
| Binary | ✅ | ✅ | ✅ | ❌ |
| HTTP semantics | ✗ (별도) | ✅ | ✅ | ✅ |
| Auto-reconnect | 직접 | n/a | n/a | ✅ |
| Proxy 호환 | 일부 X | ✅ | UDP 차단 가능 | ✅ |
| Browser 지원 | 모두 | 모두 | 모던 | 모두 (옛 X 일부) |

### WebSocket over HTTP/2 (RFC 8441)
- HTTP/2 stream 위 WebSocket
- 한 TCP 의 여러 stream 중 일부가 WS
- 일부 브라우저 / 서버

### WebTransport
- HTTP/3 위 새 API
- WebSocket 의 후속
- Chrome / Edge

---

## 12. 함정

### 함정 1 — Browser 의 보안
- `Origin` 헤더 검증 (서버) — Cross-origin WS 차단
- `wss://` 만 (모던)

### 함정 2 — Sticky Session
LB 가 한 사용자를 같은 인스턴스에. 연결 유지 필요.

### 함정 3 — Proxy 호환
일부 corporate proxy 가 WebSocket 차단. Fallback (long-polling) 필요.

### 함정 4 — Memory
한 연결당 메모리 buffer. 동시 100K+ 연결은 큰 서버.

### 함정 5 — Backpressure
서버가 빠르게 send → 클라 처리 못 함 → buffer 쌓임. Flow control.

### 함정 6 — Ping / Pong
응용이 ping 구현 안 함 → idle close (LB / Proxy).

### 함정 7 — Reconnect 의 backoff
끊김 → 즉시 재연결 → 서버 부하. Exponential backoff + jitter.

---

## 13. WebSocket 위 프로토콜

### Socket.IO
- WebSocket + long-polling fallback
- Auto-reconnect / rooms / namespaces
- Node.js 표준

### STOMP
- 메시지 큐 프로토콜 (AMQP 같은)
- WebSocket 위에서도

### SignalR
- Microsoft .NET
- WebSocket + fallback

### MQTT over WebSocket
- IoT
- Mosquitto / HiveMQ

---

## 14. 학습 자료

- **RFC 6455** (WebSocket)
- **RFC 8441** (WebSocket over HTTP/2)
- MDN WebSocket
- "Building a Realtime app with WebSockets"

---

## 15. 관련

- [[streaming]] — Streaming hub
- [[server-sent-events]] — 단방향 대안
- [[../methods/options]] — Upgrade 의 Connection
- [[../../rpc-messaging/rpc-messaging]] — WebSocket / Socket.IO
