---
title: "HTTP/1.0 — 1996 RFC 1945"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:11:00+09:00
tags:
  - network
  - http
  - http-1-0
---

# HTTP/1.0 — 1996 RFC 1945

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 헤더 / 상태 / MIME / 한계 |

**[[versions|↑ versions]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

1996년 RFC 1945 — 첫 표준 HTTP. **헤더 / 상태 코드 / MIME 도입**. 1990년대 초
웹 폭발의 기반.

---

## 2. 1995-1996 — 표준화 과정

- 1994 Mosaic / Netscape — 비공식 HTTP 사용
- 1995 IETF HTTP Working Group
- 1996.05 **RFC 1945** — Informational (정식 표준 X)
- 동시에 HTTP/1.1 작업 진행 (1.1 이 곧 표준)

---

## 3. 요청 / 응답 형식

### 요청
```http
GET /index.html HTTP/1.0
User-Agent: Mozilla/4.0
Accept: */*

```

### 응답
```http
HTTP/1.0 200 OK
Date: Mon, 13 May 1996 12:00:00 GMT
Server: NCSA/1.4
Content-Type: text/html
Content-Length: 1234
Last-Modified: Sun, 12 May 1996 10:00:00 GMT

<HTML>...
```

→ 매 요청마다 **새 TCP 연결**.

---

## 4. 0.9 대비 추가

### 4.1 메서드 — 3 개

| 메서드 | 의미 |
| --- | --- |
| **GET** | 자원 조회 |
| **POST** | 자원 처리 / 생성 |
| **HEAD** | 응답 헤더만 |

### 4.2 상태 코드

| 분류 | 예 |
| --- | --- |
| 2xx | 200 OK, 201 Created, 204 No Content |
| 3xx | 301 Moved, 302 Found, 304 Not Modified |
| 4xx | 400 Bad Request, 401 Unauth, 403 Forbidden, 404 Not Found |
| 5xx | 500 Internal, 501 Not Implemented |

### 4.3 헤더

#### General
- Date, MIME-Version, Pragma

#### Request
- User-Agent
- Referer (`Referrer` 의 misspelling 이 표준화)
- If-Modified-Since
- Authorization

#### Response
- Server
- WWW-Authenticate
- Location (3xx)
- Last-Modified
- Expires

#### Entity
- Content-Type
- Content-Length
- Content-Encoding
- Allow

### 4.4 MIME — RFC 2045 의 일부

- `Content-Type: text/html; charset=ISO-8859-1`
- `Content-Type: image/jpeg`
- `Content-Type: application/x-www-form-urlencoded`

→ HTML 외 모든 컨텐츠 가능.

### 4.5 인증

```http
GET /private/ HTTP/1.0
Authorization: Basic dXNlcjpwYXNz

→ user:pass 의 Base64
```

---

## 5. 한계

### 5.1 연결 — 매 요청 새 TCP

```
1 페이지 = HTML + 이미지 5 + CSS 2 + JS 3 = 11 요청
→ 11 × (TCP handshake + slow start) = 큰 latency
```

#### 비공식 Keep-Alive

```http
Connection: Keep-Alive
```

→ 1.1 의 표준화로 흡수.

### 5.2 Host 헤더 없음

한 IP 의 한 서버 — 한 도메인만 호스팅 가능:
```
서버 IP 1.2.3.4
→ www.example.com 만 호스팅 (URL 의 host 부분 무시됨)
```

→ Virtual Hosting 불가. 1.1 의 **Host 헤더** 가 해결.

### 5.3 캐싱 제한적

- Expires 만 (절대 시간) — 시계 동기 문제
- 1.1 의 Cache-Control 가 해결

### 5.4 Pipelining 없음

순차 — Round Trip 폭발.

### 5.5 Chunked transfer 없음

Content-Length 미리 알아야 — 동적 컨텐츠 어려움.

---

## 6. 1.0 의 살아남은 점

- **기본 메서드 / 상태 코드 / 헤더 구조** — 1.1, 2, 3 모두 계승
- **MIME** — 모든 컨텐츠 표현
- **인증 모델** — Basic, Digest 의 기반

→ 1.1 은 1.0 의 확장 (Keep-Alive 와 Host 가 핵심).

---

## 7. 함정 (현대)

- 모던 클라이언트는 거의 1.0 사용 X
- 라이브러리 / 옛 임베디드 / IoT 가 가끔
- 1.0 응답의 **Content-Length 없음** + Keep-Alive — 응답 끝 모호

---

## 8. tcpdump / curl 로 1.0 보기

```bash
curl --http1.0 -v https://example.com/
# Headers:
# > GET / HTTP/1.0
# > Host: example.com    ← 1.0 도 Host 보낼 수는 있음
# > User-Agent: curl/...
```

---

## 9. 학습 자료

- **RFC 1945** (HTTP/1.0)
- **HTTP: The Definitive Guide** Ch.1-3
- W3C "HTTP/1.0 Spec"

---

## 10. 관련

- [[http-0-9]] — 이전
- [[http-1-1]] — 후속
- [[versions]] — 버전 hub
