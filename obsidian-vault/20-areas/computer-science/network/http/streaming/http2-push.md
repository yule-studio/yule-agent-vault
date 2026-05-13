---
title: "HTTP/2 Server Push (deprecated)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T01:25:00+09:00
tags:
  - network
  - http
  - streaming
  - server-push
---

# HTTP/2 Server Push (deprecated)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 작동 / 왜 사장 / 103 Early Hints 대체 |

**[[streaming|↑ Streaming]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

서버가 클라가 요청하기 전 **미리 자원 push**. HTTP/2 의 야망 — 2022 Chrome 제거.
**103 Early Hints** 가 대체.

---

## 2. 동기

```
브라우저 / HTML 요청:
GET /index.html

응답: HTML
→ HTML 파싱 후 /style.css, /script.js 발견
→ 추가 요청 (RTT 낭비)
```

### Push 의 아이디어

```
GET /index.html
↓
서버: HTML + style.css + script.js 미리 push
→ 추가 RTT 절약
```

---

## 3. 동작 — HTTP/2 frame

```
1. Client → Server: HEADERS frame (GET /index.html)
2. Server → Client: PUSH_PROMISE frame
   - Stream ID 4 (예시)
   - Header: GET /style.css
3. Server → Client: HEADERS frame (Stream 4 의 응답 헤더)
4. Server → Client: DATA frame (Stream 4 의 body)
5. Server → Client: HEADERS frame (Stream 2 의 index.html 응답)
6. Server → Client: DATA frame (Stream 2)
```

### Stream ID
- Server-initiated stream — 짝수 (2, 4, 6, ...)
- Client-initiated stream — 홀수

---

## 4. 서버 구현 (Nginx 예)

```nginx
location = /index.html {
    http2_push /style.css;
    http2_push /script.js;
    http2_push /logo.png;
}
```

### Link header (RFC 5988)
```http
HTTP/1.1 200 OK
Link: </style.css>; rel=preload; as=style
Link: </script.js>; rel=preload; as=script
```

→ Nginx 가 preload link 보면 자동 push (모듈에 따라).

---

## 5. 왜 사장

### 5.1 캐시 hit / miss 예측 어려움
- 클라가 이미 캐시 보유 시 push = 낭비
- 서버는 클라 캐시 모름
- 잘못된 push = 대역폭 낭비

### 5.2 브라우저 / 라이브러리 의 한계
- Cache 와 push 의 상호작용 복잡
- Chrome 의 측정: 평균적으로 성능 ↑ X (또는 ↓)

### 5.3 응용 코드 / 인프라 복잡성
- 어느 자원 push? — 응용이 결정 어려움
- CDN 의 push 통과 — 일부만

### 5.4 Chrome 2022 제거
- HTTP/2 Server Push 제거 (Chrome 106)
- Firefox 등도 점진 제거
- **103 Early Hints** 로 대체

---

## 6. 103 Early Hints (RFC 8297) — 대체

자세히 → [[../status-codes/1xx-informational#103 Early Hints]]

```http
GET / HTTP/1.1

HTTP/1.1 103 Early Hints
Link: </style.css>; rel=preload; as=style
Link: </script.js>; rel=preload; as=script

[처리 중...]

HTTP/1.1 200 OK
Content-Type: text/html
[HTML]
```

### 차이
- **103** — 클라가 자원 요청 (캐시 hit 면 안 요청)
- **Push** — 서버가 강제 push (캐시 무관)

→ 103 이 클라 캐시 존중 + 효율적.

### 도입
- Cloudflare, Fastly, Shopify 채택
- Chrome / Edge 지원

---

## 7. 브라우저 별 지원 (2024)

| 브라우저 | HTTP/2 Push |
| --- | --- |
| Chrome / Edge | 제거 (106+) |
| Firefox | 점진 제거 |
| Safari | 지원 (옛) |

→ 사실상 사용 X.

---

## 8. RFC 9113 (HTTP/2 2022) — Push 유지

- 스펙은 그대로
- 실제 브라우저 / 서버 — 제거 / 무시

---

## 9. 함정

### 함정 1 — Push 로 최적화 시도
효과 없거나 역효과. 103 Early Hints 사용.

### 함정 2 — Link preload 의 자동 Push
일부 서버 — Link rel=preload 보면 자동 push. 의도 X. 명시적 제어.

### 함정 3 — Push 의 캐시 영향
캐시된 자원도 push — 대역폭 낭비.

---

## 10. 학습 자료

- **RFC 9113** Section 5.3, 8.2 (HTTP/2 Push)
- "Server Push Considered Harmful?" — Hooman Beheshti
- "Removing HTTP/2 Push" — Chrome 발표
- **RFC 8297** (103 Early Hints) 비교

---

## 11. 관련

- [[streaming]] — Streaming hub
- [[../status-codes/1xx-informational]] — 103 Early Hints
- [[../versions/http-2]] — HTTP/2
- [[../performance/performance]]
