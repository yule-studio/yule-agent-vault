---
title: "HTTP/0.9 — 1991 첫 HTTP"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:08:00+09:00
tags:
  - network
  - http
  - http-0-9
---

# HTTP/0.9 — 1991 첫 HTTP

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 역사적 의의 + 한계 |

**[[versions|↑ versions]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

1989-1991 Tim Berners-Lee 가 CERN 에서 만든 **WWW 의 최초 프로토콜**. 표준 RFC
없음. "GET / 만 있는 HTTP" — 헤더도 상태 코드도 없음.

---

## 2. 1989 Tim Berners-Lee 의 제안

- 1989 CERN — "Information Management: A Proposal"
- 1990 첫 구현 (NeXT computer, `WorldWideWeb` 브라우저)
- 1991.08.06 — 첫 공개 (alt.hypertext usenet)
- 1991 부터 "HTTP/0.9" 라고 불림 (1.0 등장 후 소급 명명)

---

## 3. 프로토콜 — 전체

```
요청:
GET /index.html\r\n

응답:
<HTML>
<TITLE>Hello</TITLE>
<BODY>Hello, World!</BODY>
</HTML>

(연결 종료)
```

### 특징
- **메서드** — GET 만
- **헤더** — 없음
- **상태 코드** — 없음 (요청 실패는 응답 없음)
- **본문** — HTML 만 (Content-Type 협상 X)
- **연결** — 매 요청마다 새 TCP, 응답 후 종료

---

## 4. 동작 (Telnet 으로 재현)

```bash
$ telnet info.cern.ch 80
GET /
<HTML>
<HEAD><TITLE>...</TITLE></HEAD>
<BODY>...</BODY>
</HTML>
$
```

→ 사실상 모든 모던 클라이언트는 0.9 지원 X. info.cern.ch 도 모던 HTTP.

---

## 5. 한계 → 1.0 의 동기

| 한계 | 1.0 의 해결 |
| --- | --- |
| HTML 만 | Content-Type 헤더 (이미지, JSON, XML, ...) |
| GET 만 | POST, HEAD 추가 |
| 상태 코드 X | 200/404/500 등 |
| 연결 정보 X | Date, Server, Last-Modified |
| 인증 X | WWW-Authenticate (Basic) |
| 캐싱 X | Expires, Last-Modified |

→ 1996 RFC 1945 (HTTP/1.0) 로 발전.

---

## 6. 역사적 의의

- **하이퍼링크** 개념 — `<a href>` 의 본질
- **URI** — Uniform Resource Identifier
- **클라이언트-서버** 모델 (인터넷에 보편)
- **stateless** — 매 요청 독립 (확장성의 기반)
- **MIT License-like** 공개 — 인터넷 폭발의 발판

---

## 7. 함정 (현대 의의)

- HTTP/0.9 응답은 **헤더 없음** → 어디서 응답 끝났는지 본문 자체로 판단
- 모던 서버 (Apache, Nginx) 가 0.9 응답을 보낼 위험 (옛 CGI / proxy)
- 보안 — "HTTP/0.9 response splitting" 공격 (CRLF injection)

→ 모던 서버는 0.9 응답 차단 (Apache: `HttpProtocolOptions Strict`).

---

## 8. 학습 자료

- Tim Berners-Lee — "Information Management: A Proposal" (1989)
- W3C — "HTTP/0.9 Spec" 회고
- "Web at 25" — CERN 25주년 자료
- info.cern.ch — 첫 웹사이트 복원

---

## 9. 관련

- [[http-1-0]] — 발전
- [[versions]] — 버전 hub
