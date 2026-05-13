---
title: "SSE — Server-Sent Events"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T08:18:00+09:00
tags:
  - network
  - sse
  - http
  - real-time
---

# SSE — Server-Sent Events

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | EventSource / 단방향 push |

**[[rpc-messaging|↑ RPC/Messaging]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **표준** | HTML5 (WHATWG) |
| **Content-Type** | text/event-stream |
| **방향** | 서버 → 클라이언트 (단방향) |
| **재연결** | 자동 (Last-Event-ID) |

---

## 1. 한 줄 정의

**HTTP 위의 서버 push** — 한 GET 요청, 응답이 stream 으로 무한히. 텍스트 only, 자동 재연결.

---

## 2. 동작

```
Client: GET /events HTTP/1.1
        Accept: text/event-stream

Server: HTTP/1.1 200 OK
        Content-Type: text/event-stream
        Cache-Control: no-cache
        Connection: keep-alive
        
        data: hello\n\n
        data: world\n\n
        
        event: update\n
        data: {"id": 1}\n\n
        
        id: 42\n
        data: with-id\n\n
        
        retry: 5000\n
        data: ...\n\n
        
        (계속)
```

### 메시지 포맷
- `data: ...` — 메시지 데이터
- `event: ...` — 이벤트 이름 (선택)
- `id: ...` — 이벤트 ID (재연결 시 사용)
- `retry: ...` — 재연결 간격 (ms)
- 빈 줄 — 메시지 구분
- `\n` 또는 `\r\n` 또는 `\r`

---

## 3. 브라우저 — EventSource

```javascript
const source = new EventSource('/events');

source.onopen = () => console.log('Connected');
source.onerror = (e) => console.error(e);

source.onmessage = (e) => {
    console.log('Message:', e.data);
};

// Named event
source.addEventListener('update', (e) => {
    const data = JSON.parse(e.data);
    console.log('Update:', data);
});

// 종료
source.close();
```

### 자동 재연결
- 끊기면 자동 재연결
- Last-Event-ID 헤더 — 마지막 받은 ID
- Server — 그 이후부터 재전송

---

## 4. 서버 — Node.js

```javascript
app.get('/events', (req, res) => {
    res.writeHead(200, {
        'Content-Type': 'text/event-stream',
        'Cache-Control': 'no-cache',
        'Connection': 'keep-alive',
        'Access-Control-Allow-Origin': '*'
    });
    
    // 첫 메시지
    res.write('data: connected\n\n');
    
    // 주기적
    const interval = setInterval(() => {
        const msg = JSON.stringify({ time: Date.now() });
        res.write(`data: ${msg}\n\n`);
    }, 1000);
    
    req.on('close', () => clearInterval(interval));
});
```

---

## 5. 서버 — Python (Flask)

```python
from flask import Response, stream_with_context
import time, json

@app.route('/events')
def events():
    def generate():
        while True:
            data = json.dumps({'time': time.time()})
            yield f'data: {data}\n\n'
            time.sleep(1)
    return Response(stream_with_context(generate()), mimetype='text/event-stream')
```

---

## 6. SSE vs WebSocket

| | SSE | WebSocket |
| --- | --- | --- |
| **방향** | 서버 → 클라 | 양방향 |
| **포맷** | text only | text / binary |
| **HTTP** | 위에 있음 | HTTP 시작, 그 후 raw |
| **재연결** | 자동 (Last-Event-ID) | 수동 |
| **방화벽** | 보통 HTTP — OK | HTTPS 대부분 OK |
| **부하** | 적음 | 적음 |
| **복잡** | 단순 | 보통 |
| **사용** | 알림 / 라이브 피드 | 채팅 / 게임 |

### SSE 선택
- 단방향이면 충분
- HTTP infrastructure 친화
- 자동 재연결 원함

### WebSocket 선택
- 양방향 필요
- Binary 데이터
- Low latency

---

## 7. ChatGPT / LLM streaming — SSE

### 사례
- OpenAI / Anthropic / 대부분 LLM API — SSE 로 token stream
- 응답이 점진적 출력

```
Client: POST /v1/chat/completions { stream: true }

Server:
data: {"choices": [{"delta": {"content": "Hello"}}]}
data: {"choices": [{"delta": {"content": ", "}}]}
data: {"choices": [{"delta": {"content": "world"}}]}
data: [DONE]
```

→ 단방향, 자동 재연결, HTTP 위 — SSE 가 최적.

---

## 8. SSE 의 한계 / 함정

### 함정 1 — Buffering
Nginx / Cloudflare 의 응답 buffer → 즉시 전달 X.
`X-Accel-Buffering: no` 헤더 / proxy_buffering off.

### 함정 2 — 연결 수 한계
브라우저 — 한 origin 의 동시 connection 한계 (~6 HTTP/1.1).
HTTP/2 — multiplexing 으로 해결.

### 함정 3 — Idle timeout
LB / Proxy 의 timeout. Keep-alive comment 주기.
```
: ping\n\n      ← comment (무시되는 keep-alive)
```

### 함정 4 — IE / 옛 브라우저
EventSource API — IE / Edge 옛 버전 X. polyfill 필요.

### 함정 5 — UTF-8 only
text/event-stream — 텍스트만. binary 는 base64.

### 함정 6 — Last-Event-ID 의 storage
재연결 시 — 서버가 이전 메시지 알아야. 큰 cluster — Redis / Kafka.

### 함정 7 — Cors
브라우저 — `withCredentials` 옵션 / CORS 헤더.

---

## 9. SSE 의 모던 활용

### LLM streaming
- OpenAI / Anthropic / Gemini

### Notification feed
- GitHub web notifications
- Twitter timeline 일부

### Live updates
- 주가 / 스코어
- 댓글 / 좋아요

### 옛 — Long polling 대체
- 사용 ↓ — WebSocket 또는 SSE

---

## 10. 학습 자료

- HTML Living Standard — EventSource
- "Server-Sent Events" (Eric Bidelman)
- OpenAI Streaming docs (SSE 예시)

---

## 11. 관련

- [[rpc-messaging]] — Hub
- [[websocket]] — 비교
- [[../http/versions/http-2]] — multiplexing 으로 connection limit 해결
