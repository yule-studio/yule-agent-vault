---
title: "Long Polling (옛 비동기 패턴)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T01:30:00+09:00
tags:
  - network
  - http
  - streaming
  - long-polling
---

# Long Polling (옛 비동기 패턴)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Short / Long Polling / SSE/WS 의 대안 |

**[[streaming|↑ Streaming]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

WebSocket / SSE 없이 **HTTP 만으로 push 와 비슷한 효과**. 옛 비동기 패턴.

---

## 2. 3 가지 폴링

### 2.1 Short Polling

```javascript
setInterval(async () => {
    const r = await fetch('/api/messages?since=' + lastId);
    const data = await r.json();
    // ...
}, 1000);
```

→ 1 초마다 요청. **단순 / 비효율**.

### 2.2 Long Polling

```javascript
async function poll() {
    while (true) {
        try {
            const r = await fetch('/api/messages?since=' + lastId);
            const data = await r.json();
            // 처리
            lastId = data.lastId;
        } catch (e) {
            await sleep(1000);   // 오류 시 대기
        }
    }
}
```

→ 서버가 **데이터 있을 때까지 응답 대기** (즉시 안 보냄).

### 2.3 Comet
- 옛 용어 — Long Polling 등 비동기 패턴 통칭
- 2006-2010 인기, WebSocket 등장 후 사장

---

## 3. Long Polling 의 서버 측

```python
@app.get("/api/messages")
async def messages(since: int = 0):
    timeout = 30
    start = time.time()
    while time.time() - start < timeout:
        new = db.query(Message).filter(id__gt=since).all()
        if new:
            return {"messages": new, "lastId": new[-1].id}
        await asyncio.sleep(0.5)    # 폴링
    return {"messages": [], "lastId": since}
```

### 흐름
1. 클라 → 서버: GET (since=X)
2. 서버: 새 데이터 있나? 없으면 대기
3. 새 데이터 도착 또는 timeout → 응답
4. 클라 → 다시 요청

---

## 4. Long Polling vs SSE vs WebSocket

| 측면 | Short Poll | Long Poll | SSE | WebSocket |
| --- | --- | --- | --- | --- |
| Latency | 평균 0.5s+ | 즉시 (이벤트) | 즉시 | 즉시 |
| 서버 부하 | 매 N 초 | 한 연결 | 한 연결 | 한 연결 |
| 클라 코드 | 단순 | 중 | 단순 (API) | 직접 |
| 양방향 | 별도 호출 | 별도 호출 | ❌ | ✅ |
| Auto-reconnect | 자동 (poll) | 자동 (poll) | ✅ | 직접 |
| HTTP semantics | ✅ | ✅ | ✅ | ✗ |
| Proxy 호환 | ✅ | ✅ | ✅ | 일부 X |

→ **Long Polling 거의 X — SSE / WebSocket 권장**.

---

## 5. 현재 사용 사례

### 5.1 Fallback (WebSocket 차단 환경)
- Socket.IO 가 자동 fallback
- 회사 / 모바일 망의 proxy 가 WS 차단 시

### 5.2 옛 호환성
- IE 8 등 EventSource / WebSocket X

### 5.3 단순한 알림
- 작은 응용 — 그냥 long poll 도 OK

---

## 6. 함정

### 함정 1 — Timeout 너무 김
60 초 = LB / Proxy timeout. 25-30 초 권장.

### 함정 2 — Connection 누적
모든 사용자가 long poll → 서버 한 인스턴스의 최대 connection 한계.

### 함정 3 — 사용자별 폴링 빈도
이벤트 흐릿 시 폴링 폭증.

### 함정 4 — 재시도 폭주
서버 죽으면 모든 클라 동시 재시도 → 폭증. Exponential backoff.

### 함정 5 — 데이터 누락
재시도 사이 발생한 이벤트 — `since=lastId` 로 보존.

---

## 7. 학습 자료

- "Comet" — 옛 Wikipedia
- "Long polling vs WebSocket vs SSE"
- Socket.IO transport fallback

---

## 8. 관련

- [[streaming]] — Streaming hub
- [[websocket]] — 양방향 권장
- [[server-sent-events]] — 단방향 권장
- [[chunked-transfer]]
