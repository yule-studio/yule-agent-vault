---
title: "1xx — Informational"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:13:00+09:00
tags:
  - network
  - http
  - status-codes
  - 1xx
---

# 1xx — Informational

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 100/101/102/103 |

**[[status-codes|↑ Status Codes]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

"처리 중 — 더 기다려" 의미. 임시 응답 — 최종 응답은 별도. HTTP/1.0 에는 없었음
(1.1 도입).

---

## 2. 표준 코드

### 100 Continue

```http
PUT /upload HTTP/1.1
Content-Length: 1073741824    (1 GB)
Expect: 100-continue

[body 안 보냄, 대기]

HTTP/1.1 100 Continue
[클라가 body 송신 시작]

HTTP/1.1 201 Created
...

OR:

HTTP/1.1 417 Expectation Failed
[클라가 body 송신 X]
```

#### 동기
큰 body 의 사전 검증:
- 인증 실패 / 권한 X / 자원 한계
- 1 GB 보내고 거부받는 것보다 사전 거부

#### Expect: 100-continue
- 클라가 명시 헤더 보냄
- 서버가 100 또는 4xx 응답
- 일부 라이브러리는 자동 (curl, libcurl)

#### 함정
- 일부 미들박스가 Expect 헤더 폐기
- 대부분 환경에서 잘 안 씀 — 모던은 chunked upload + 응용 검증

### 101 Switching Protocols

```http
GET /chat HTTP/1.1
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Key: ...

HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Accept: ...

[이후 WebSocket 프로토콜]
```

#### 사용
- **WebSocket** — 가장 흔함
- HTTP/2 의 h2c (cleartext) — 거의 사용 X
- 일부 HTTP/3 시그널링

자세히 → [[../streaming/websocket]]

### 102 Processing (deprecated)

```http
HTTP/1.1 102 Processing
```

WebDAV — "처리 중이지만 시간 걸림". 클라 timeout 방지.

→ **2018 WebDAV WG 가 deprecated** — 잘 안 씀. SSE / WebSocket / long-polling 대체.

### 103 Early Hints (RFC 8297)

```http
GET / HTTP/1.1

HTTP/1.1 103 Early Hints
Link: </style.css>; rel=preload; as=style
Link: </script.js>; rel=preload; as=script

[처리 계속...]

HTTP/1.1 200 OK
Content-Type: text/html
...
```

#### 동기
- 본 응답 대기 중 클라가 자원 미리 fetch
- HTTP/2 Server Push 의 대체 (Push 가 사장됨)

#### 사용
- Chrome / Edge 지원 (2022+)
- Fastly, Cloudflare CDN 채택
- Shopify 가 큰 성능 개선 보고

#### 흐름
```
1. 서버가 DB 쿼리 (느림)
2. 그동안 103 + Link 헤더로 정적 자원 hint
3. 브라우저가 정적 자원 fetch 시작
4. DB 완료 → 200 OK + HTML
5. HTML 이 정적 자원 사용 — 이미 캐시됨
```

---

## 3. HTTP/1.0 vs 1.1

HTTP/1.0 에는 1xx 없음:
- 100 Continue 도 1.1 도입
- 옛 클라이언트 (1.0) 에는 1xx 보내면 안 됨

---

## 4. 응용 처리

### 4.1 라이브러리 자동 처리
- curl: `Expect: 100-continue` 자동 (큰 body)
- requests (Python): 명시 X
- fetch (JS): 자동

### 4.2 받아도 응용 코드 안 봄
- HTTP 라이브러리가 1xx 받고 다음 응답 기다림
- 최종 응답만 응용에 노출

### 4.3 103 Early Hints 의 예외
- 브라우저는 1xx 도 처리 (link prefetch)
- 응용 코드는 보통 200+ 만

---

## 5. 함정

### 함정 1 — Expect: 100-continue 의 안 받음
일부 서버 / 미들박스가 무시 → 클라가 timeout 후 body 보냄 (정상이지만 느림).

### 함정 2 — WebSocket 의 101 외 응답
업그레이드 거부 시 400 Bad Request — 클라가 fallback 어려움.

### 함정 3 — 102 Processing 사용
deprecated — 사용 X. 비동기는 202 Accepted + polling.

### 함정 4 — 103 의 미지원 클라
호환 X — 일반 200 처럼 보임 (1xx 무시). 안전.

---

## 6. 학습 자료

- **RFC 9110** Section 15.2
- **RFC 8297** (103 Early Hints)
- RFC 8470 (Using Early Data) — 0-RTT 와 결합
- Cloudflare blog — Early Hints

---

## 7. 관련

- [[status-codes]] — hub
- [[../streaming/websocket]] — 101
- [[../performance/performance]] — 103 으로 최적화
- [[2xx-success]]
