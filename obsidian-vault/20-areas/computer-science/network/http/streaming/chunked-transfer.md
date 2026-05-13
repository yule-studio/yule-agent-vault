---
title: "Chunked Transfer Encoding"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T01:10:00+09:00
tags:
  - network
  - http
  - streaming
  - chunked
---

# Chunked Transfer Encoding

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 형식 / 사용 / HTTP/2 변화 |

**[[streaming|↑ Streaming]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

응답 본문을 **size + data 의 chunk** 들로 송신. Content-Length 미리 알 필요 X.
HTTP/1.1 의 스트리밍 메커니즘.

---

## 2. 형식

```http
HTTP/1.1 200 OK
Content-Type: text/html
Transfer-Encoding: chunked

7\r\n
Mozilla\r\n
9\r\n
Developer\r\n
7\r\n
Network\r\n
0\r\n
\r\n
```

### 각 chunk
```
[hex size]\r\n
[data]\r\n
```

### 끝
```
0\r\n
\r\n
```

→ `0` size 의 chunk = 끝.

---

## 3. 동기

### Content-Length 미리 알 필요 X
- 동적 콘텐츠 (DB 쿼리 후 생성)
- Streaming (계속 추가)
- 큰 파일 (전체 읽기 전 응답)

### 옛 방법 (Content-Length)
```
1. 전체 응답 생성
2. 크기 계산
3. Content-Length 헤더
4. 송신
```

→ 메모리 모두 사용 + 첫 byte 지연.

### Chunked
```
1. 데이터 생성 시작
2. 일부 chunk → 송신
3. 다음 chunk → ...
4. 끝 0\r\n
```

→ 메모리 효율 + 첫 byte 빠름.

---

## 4. Trailer 헤더

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Trailer: ETag, Server-Timing

5\r\n
Hello\r\n
0\r\n
ETag: "v1"\r\n
Server-Timing: db;dur=53\r\n
\r\n
```

### 의미
- 응답 시작 시 모르는 값을 끝에 보냄
- gRPC over HTTP/2 가 status code 를 trailer 로
- 일반 HTTP — 거의 사용 X

---

## 5. 사용 예 — 큰 응답

### 동적 SSR (Server-Side Rendering)

```
1. <html><head>...</head> 송신 (즉시)
2. DB 쿼리 (1 초)
3. <body>...데이터... 송신
4. </body></html> 송신
5. 0\r\n 끝
```

→ 사용자: HTML 헤더부터 즉시 보임 (Streaming SSR).

### CSV 다운로드

```python
def export_csv():
    def generate():
        yield "id,name,email\n"
        for user in db.query(User):
            yield f"{user.id},{user.name},{user.email}\n"
    return Response(generate(), mimetype="text/csv")
```

→ 대량 데이터 메모리 X.

### Logs / Tail

```bash
curl -N https://example.com/logs/tail
# -N: --no-buffer
# 새 로그가 들어올 때마다 표시
```

---

## 6. HTTP/2 / HTTP/3 의 변화

### Transfer-Encoding 제거
- HTTP/2 — 헤더 사용 X
- Frame 이 chunking 처리
- DATA frame 의 END_STREAM flag

### 응용 코드는 같음
- 응용은 stream 처럼 write
- HTTP 라이브러리가 frame 으로 변환

---

## 7. 클라 측 처리

### fetch (JS)
```javascript
const r = await fetch('/streaming');
const reader = r.body.getReader();

while (true) {
    const {done, value} = await reader.read();
    if (done) break;
    console.log(value);    // Uint8Array
}
```

### Node.js
```javascript
const https = require('https');
https.get('https://example.com/stream', (res) => {
    res.on('data', (chunk) => {
        process.stdout.write(chunk);
    });
});
```

### Python
```python
import requests
with requests.get('https://example.com/stream', stream=True) as r:
    for chunk in r.iter_content(chunk_size=1024):
        process(chunk)
```

### curl
```bash
curl -N https://...      # --no-buffer
```

---

## 8. 서버 측 구현

### Express
```javascript
app.get('/stream', (req, res) => {
    res.setHeader('Content-Type', 'text/plain');
    // Transfer-Encoding: chunked 자동
    let count = 0;
    const interval = setInterval(() => {
        res.write(`chunk ${count++}\n`);
        if (count >= 10) {
            clearInterval(interval);
            res.end();
        }
    }, 1000);
});
```

### Flask
```python
from flask import Response

@app.route('/stream')
def stream():
    def generate():
        for i in range(10):
            yield f"chunk {i}\n"
            time.sleep(1)
    return Response(generate(), mimetype='text/plain')
```

### FastAPI
```python
from fastapi.responses import StreamingResponse

@app.get("/stream")
def stream():
    def generate():
        for i in range(10):
            yield f"chunk {i}\n"
            time.sleep(1)
    return StreamingResponse(generate(), media_type='text/plain')
```

---

## 9. Buffering — Nginx / Proxy 의 영향

### Nginx 의 기본 buffering
- 응답 전체 buffer 후 클라 송신
- chunked streaming 무력화

### 해결 — buffer off
```nginx
proxy_buffering off;        # 응답 즉시 forward
proxy_cache_bypass 1;
```

또는 응답 헤더로:
```http
X-Accel-Buffering: no
```

→ Nginx 가 buffer X.

---

## 10. 함정

### 함정 1 — Content-Length + chunked 동시
RFC 위반 + Request Smuggling 위험.

### 함정 2 — Nginx buffering
proxy_buffering off 또는 X-Accel-Buffering: no.

### 함정 3 — 응답 flush 필요
응용 코드가 write 후 flush 안 함 → 메모리 buffer.

```python
sys.stdout.flush()         # Python
res.write(...); res.flush();  # JS
```

### 함정 4 — 큰 chunk size
한 chunk 가 너무 크면 streaming 의미 없음. 작게 (KB 단위).

### 함정 5 — 클라 buffer
fetch 의 응답이 일정 크기 받아야 reader 가 yield. text/event-stream + flush 권장.

### 함정 6 — HTTP/2 의 헤더
Transfer-Encoding 헤더 보내면 RFC 위반.

### 함정 7 — Compression + Chunked
gzip → chunked → gzip 해제 → 표시. Vary: Accept-Encoding.

---

## 11. 학습 자료

- **RFC 9112** Section 7.1 (Chunked)
- "Streaming HTTP responses" — Patrick McKenzie
- "Server-Sent Events vs Chunked" 비교

---

## 12. 관련

- [[streaming]] — Streaming hub
- [[server-sent-events]] — SSE 도 chunked 위
- [[../performance/transfer-encoding]] — TE vs CE
- [[../headers/general-headers]] — Transfer-Encoding
