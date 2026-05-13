---
title: "HTTP 스트리밍 (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T01:05:00+09:00
tags:
  - network
  - http
  - streaming
---

# HTTP 스트리밍 (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Chunked / SSE / WebSocket / Server Push / Long-polling |

**[[../http|↑ HTTP]]** · **[[../../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

HTTP 의 단순 request-response 모델을 **장기간 연결 / 단방향-양방향 스트림** 으로 확장.

---

## 2. 4 가지 패턴

| 패턴 | 방향 | 프로토콜 |
| --- | --- | --- |
| **Chunked Transfer** | 서버 → 클라 (단방향) | HTTP/1.1 |
| **Server-Sent Events** | 서버 → 클라 (단방향) | HTTP + text/event-stream |
| **WebSocket** | 양방향 | HTTP Upgrade → WebSocket |
| **HTTP/2 Server Push** | 서버 → 클라 (사장) | HTTP/2 |
| **Long Polling** | "양방향" (HTTP 기반) | HTTP 일반 |

---

## 3. 비교 표

| 측면 | Chunked | SSE | WebSocket | HTTP/2 Push | Long Poll |
| --- | --- | --- | --- | --- | --- |
| 방향 | server → client | server → client | bidi | server → client | bidi |
| Protocol | HTTP | HTTP | WS | HTTP/2 | HTTP |
| Auto-reconnect | X | ✅ | 직접 | X | 직접 |
| Binary | OK | text 만 | OK | OK | OK |
| Browser API | fetch | EventSource | WebSocket | (resource) | fetch loop |
| Use case | 큰 응답 | 알림 / 진행 | chat / 게임 | (사장) | 옛 |

---

## 4. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[chunked-transfer]] | Transfer-Encoding: chunked |
| [[server-sent-events]] | SSE / EventSource |
| [[websocket]] | HTTP Upgrade → WebSocket |
| [[http2-push]] | Server Push (deprecated) |
| [[long-polling]] | 옛 비동기 패턴 |

---

## 5. 사용 결정

```
방향?
├ Server → Client (단방향)
│   ├ 짧은 응답 — 일반 HTTP
│   ├ 큰 응답 — chunked
│   ├ 이벤트 알림 (text) — SSE
│   └ 옛 / 호환성 — long polling
└ Bidirectional
    ├ Chat / 게임 / 실시간 — WebSocket
    └ 단순 — long polling
```

---

## 6. HTTP/2 / HTTP/3 의 스트리밍

### Frame 기반
- 모든 응답이 본질적 streaming (frame 단위)
- Chunked 헤더 사용 X (frame 이 처리)

### Multiplexing
- 한 연결 안 수많은 stream
- 각 stream 이 별도 streaming

### gRPC over HTTP/2
- Server streaming / Client streaming / Bidi streaming
- 자세히 → [[../../rpc-messaging/rpc-messaging]]

---

## 7. 학습 자료

- "Streams" — Web standard
- MDN — Streaming APIs
- "Server-Sent Events vs WebSocket" — comparison

---

## 8. 관련

- [[../http]] — HTTP hub
- [[chunked-transfer]] / [[server-sent-events]] / [[websocket]] / [[http2-push]] / [[long-polling]]
- [[../performance/transfer-encoding]] — chunked 의 위치
- [[../../rpc-messaging/rpc-messaging]] — gRPC / WebRTC
