---
title: "Server-Sent Events (SSE)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T01:15:00+09:00
tags:
  - network
  - http
  - streaming
  - sse
---

# Server-Sent Events (SSE)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | EventSource / text/event-stream / reconnect |

**[[streaming|↑ Streaming]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

서버 → 클라 **단방향 텍스트 이벤트 스트림**. EventSource API. `text/event-stream`
Content-Type. Auto-reconnect. W3C 표준 (HTML5).

---

## 2. 응답 형식

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

event: message
data: Hello, World!

event: notification
data: {"type":"new_message","from":"Alice"}

id: 42
event: update
data: line 1
data: line 2

retry: 10000

```

### 필드
- **data:** — 이벤트 데이터 (필수)
- **event:** — 이벤트 타입 (옵션, 기본 "message")
- **id:** — 이벤트 ID (재연결 시 Last-Event-ID 로 보냄)
- **retry:** — 재연결 timeout (ms)
- **:** — comment (heartbeat)

### 구분
- 빈 줄 (`\n\n`) — 한 이벤트 끝
- 한 이벤트에 여러 data 줄 — 줄바꿈으로 결합

---

## 3. 클라이언트 — EventSource

```javascript
const es = new EventSource('/api/events');

es.onmessage = (event) => {
    console.log(event.data);
};

es.addEventListener('notification', (event) => {
    const data = JSON.parse(event.data);
    console.log('New notification:', data);
});

es.onerror = (err) => {
    console.error('SSE error:', err);
};

// 종료
es.close();
```

### 특징
- 자동 재연결 (auto-reconnect)
- 마지막 ID 기억 (Last-Event-ID 헤더)
- 간단한 API

---

## 4. 자동 재연결

```
연결 끊김 → 3초 (기본) 후 재연결 시도
서버에 Last-Event-ID 헤더 보냄

GET /events HTTP/1.1
Last-Event-ID: 42

→ 서버: 42 이후 이벤트만 송신 (resume)
```

### retry 필드로 조정
```
retry: 5000

→ 5 초 timeout
```

---

## 5. 서버 구현

### Express
```javascript
app.get('/events', (req, res) => {
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    res.setHeader('X-Accel-Buffering', 'no');   // Nginx
    
    const lastId = req.headers['last-event-id'];
    let id = lastId ? parseInt(lastId) + 1 : 1;
    
    const interval = setInterval(() => {
        res.write(`id: ${id}\n`);
        res.write(`event: tick\n`);
        res.write(`data: ${new Date().toISOString()}\n\n`);
        id++;
    }, 1000);
    
    req.on('close', () => clearInterval(interval));
});
```

### FastAPI
```python
from fastapi.responses import StreamingResponse

@app.get("/events")
async def events(request: Request):
    async def event_generator():
        last_id = int(request.headers.get('last-event-id', 0))
        while True:
            if await request.is_disconnected():
                break
            last_id += 1
            yield f"id: {last_id}\n"
            yield f"event: tick\n"
            yield f"data: {datetime.now().isoformat()}\n\n"
            await asyncio.sleep(1)
    
    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

### Flask
```python
from flask import Response

@app.route('/events')
def events():
    def generate():
        last_id = int(request.headers.get('Last-Event-ID', 0))
        while True:
            last_id += 1
            yield f"id: {last_id}\n"
            yield f"data: {datetime.now()}\n\n"
            time.sleep(1)
    return Response(generate(), mimetype='text/event-stream')
```

---

## 6. 사용 사례

### 6.1 실시간 알림
```
GitHub notification, Slack message, 게시물 좋아요 등
```

### 6.2 진행 상태
```
파일 업로드 progress
배치 작업 status
LLM streaming (ChatGPT 등)
```

### 6.3 라이브 데이터
```
주식 시세 (단방향)
스포츠 점수
서버 metrics
```

### 6.4 LLM Streaming
```
OpenAI / Anthropic / Google API
응답 token 별 SSE
```

---

## 7. SSE vs WebSocket

| 측면 | SSE | WebSocket |
| --- | --- | --- |
| 방향 | server → client | bidi |
| Protocol | HTTP | WS (HTTP Upgrade) |
| 데이터 | text 만 | text + binary |
| Auto-reconnect | ✅ | 직접 구현 |
| Browser API | EventSource | WebSocket |
| Proxy / Firewall | OK (HTTP) | 일부 차단 |
| HTTP/2/3 | ✅ | h2c X (HTTP/2 의 별도 spec) |
| CORS | OK | 별도 |

→ 단방향 + 단순 — **SSE**. 양방향 / binary — **WebSocket**.

자세히 → [[websocket]]

---

## 8. SSE 의 장점

### 8.1 단순
- HTTP 의 일반 요청 / 응답
- 인증 / CORS / 보안 정책 그대로
- WebSocket 의 핸드셰이크 / 프레이밍 X

### 8.2 자동 재연결
- 라이브러리 / 응용 코드 X
- Last-Event-ID 도 자동

### 8.3 HTTP/2 친화
- 한 연결의 stream 으로
- WebSocket 은 별도 spec (RFC 8441)

### 8.4 단방향이 충분한 경우 많음
- 알림 / 진행 / 라이브 데이터
- 클라가 보내는 건 적음 (별도 REST 호출 OK)

---

## 9. SSE 의 한계

### 9.1 단방향
- 클라 → 서버는 별도 요청
- 양방향 chat 어려움 — WebSocket 권장

### 9.2 Text 만
- Binary 불가 (Base64 인코딩 가능하지만 비효율)

### 9.3 6 connection 한계 (HTTP/1.1)
- 도메인 당 6 — SSE 만 6 개면 다른 자원 못 받음
- **HTTP/2 사용 시 해결** (멀티 stream)

### 9.4 Browser 호환
- IE / 옛 Edge — 미지원
- Polyfill (eventsource.js)

---

## 10. Heartbeat (Connection Keep-alive)

```
일정 시간 데이터 없으면 proxy / LB 가 연결 close

해결: comment 송신 (heartbeat)
: heartbeat\n\n
```

→ 클라가 무시 (`:` = comment), 연결 유지.

---

## 11. 함정

### 함정 1 — Nginx buffering
proxy_buffering off / X-Accel-Buffering: no.

### 함정 2 — 응용 flush 누락
write 후 flush — 즉시 송신.

### 함정 3 — HTTP/1.1 의 6 connection
SSE 가 모두 차지 → 다른 자원 X. HTTP/2.

### 함정 4 — Auth Token
EventSource API 는 헤더 명시 X — URL query 또는 Cookie 만.
```javascript
new EventSource('/events?token=abc');   // URL
// 또는 Cookie (자동)
```

### 함정 5 — CORS + EventSource
```javascript
new EventSource('/events', {withCredentials: true});
```
- 서버에 Allow-Credentials + 명시 Origin

### 함정 6 — Mobile 의 background
앱이 background 시 OS 가 연결 끊음. 재연결 의존.

### 함정 7 — 큰 데이터
한 이벤트가 너무 크면 메모리 부담. 작게 나눔.

---

## 12. 학습 자료

- **HTML Living Standard** — Server-sent events
- MDN EventSource / SSE
- "Streaming responses with HTTP/2 SSE" — Cloudflare

---

## 13. 관련

- [[streaming]] — Streaming hub
- [[websocket]] — 양방향
- [[chunked-transfer]] — SSE 도 chunked 위
- [[long-polling]] — 옛 대안
