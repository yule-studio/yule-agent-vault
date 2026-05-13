---
title: "Transfer-Encoding vs Content-Encoding"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:55:00+09:00
tags:
  - network
  - http
  - performance
  - transfer-encoding
---

# Transfer-Encoding vs Content-Encoding

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | hop-by-hop vs end-to-end |

**[[performance|↑ Performance]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 두 헤더의 차이

| 측면 | Transfer-Encoding | Content-Encoding |
| --- | --- | --- |
| **적용** | hop-by-hop | end-to-end |
| **의미** | 전송 방식 | 자원 자체 |
| **HTTP/2/3** | 제거 (frame 처리) | 유지 |
| **재계산** | hop 마다 가능 | 변경 X |
| **ETag** | 영향 X | 다름 |

---

## 2. Content-Encoding (end-to-end)

```http
HTTP/1.1 200 OK
Content-Type: text/html
Content-Encoding: gzip
Content-Length: 2500
```

- 자원 자체가 gzip 압축
- 클라가 압축 해제 → text/html
- 모든 hop 동일

자세히 → [[compression-encoding]]

---

## 3. Transfer-Encoding (hop-by-hop)

```http
HTTP/1.1 200 OK
Content-Type: text/html
Transfer-Encoding: chunked
```

- 다음 hop 에 chunked 로 전송
- 다음 hop 이 다시 해석 / 재인코딩 가능
- 마지막 클라가 받을 때는 다른 형식일 수 있음

---

## 4. chunked — 가장 흔한 Transfer-Encoding

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Content-Type: text/html

4\r\n
Hell\r\n
3\r\n
o W\r\n
5\r\n
orld!\r\n
0\r\n
\r\n
```

### 의미
- 본문을 여러 chunk 로 분할
- 각 chunk = size + data
- `0\r\n\r\n` = 끝

자세히 → [[../streaming/chunked-transfer]]

---

## 5. Content-Length vs Transfer-Encoding

### 둘 중 하나만 (RFC)
```http
Content-Length: 1234        ← 미리 알 때
또는
Transfer-Encoding: chunked  ← 모를 때 / 스트리밍
```

### 둘 다 = Request Smuggling 위험
```http
Content-Length: 100
Transfer-Encoding: chunked
```

→ Proxy / 서버가 다른 헤더 우선 — 큰 보안 위험.

자세히 → [[../../tcp/tcp-attacks]] (Request Smuggling)

---

## 6. Transfer-Encoding 의 다른 값

```http
Transfer-Encoding: gzip, chunked
Transfer-Encoding: deflate
```

### 사용
- 거의 X — Content-Encoding 가 표준
- 일부 옛 응용

---

## 7. HTTP/2/3 의 변화

### Transfer-Encoding 제거
- HTTP/2 frame 이 chunking 처리
- `Transfer-Encoding: chunked` 헤더 보내면 RFC 위반

### Content-Encoding 유지
- 응답 자체의 인코딩

---

## 8. TE 헤더 (요청 측)

```http
GET / HTTP/1.1
TE: trailers, gzip
```

→ 클라가 받을 수 있는 Transfer-Encoding 명시. 거의 X.

---

## 9. Hop-by-hop 헤더 일반

RFC 9110 의 hop-by-hop:
- Connection
- Keep-Alive
- Proxy-Authenticate / Proxy-Authorization
- TE
- Trailer
- Transfer-Encoding
- Upgrade

→ Proxy 가 forward 시 제거 (또는 자기 값으로 변경).

End-to-end 는 그대로 forward.

---

## 10. 함정

### 함정 1 — TE / CE 혼동
- Transfer-Encoding = 전송 시
- Content-Encoding = 자원 자체

### 함정 2 — Request Smuggling
Content-Length + Transfer-Encoding 동시 — 절대 X.

### 함정 3 — HTTP/2 의 TE
Frame 이 처리 — 헤더로 보내면 거부.

### 함정 4 — Chunked 의 마지막 0
`0\r\n\r\n` 빠뜨림 → 응답 stuck.

### 함정 5 — Proxy 의 chunked → length
일부 Proxy 가 chunked 응답을 받아 Content-Length 로 변환 — 일관성 깨질 수 있음.

---

## 11. 학습 자료

- **RFC 9112** Section 7 (Transfer Coding)
- **RFC 9110** Section 8.4 (Content-Encoding)
- "HTTP/1.1: Transfer Coding" — explanation

---

## 12. 관련

- [[performance]] — Performance hub
- [[compression-encoding]] — Content-Encoding 깊이
- [[../streaming/chunked-transfer]] — chunked 깊이
- [[../headers/general-headers]] — Transfer-Encoding 위치
