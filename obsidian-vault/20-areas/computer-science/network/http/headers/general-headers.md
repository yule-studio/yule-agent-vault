---
title: "General Headers — 요청/응답 공통"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:40:00+09:00
tags:
  - network
  - http
  - headers
  - general
---

# General Headers — 요청/응답 공통

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Date / Connection / Cache-Control / Via / Pragma / Trailer |

**[[headers|↑ Headers]]** · **[[../http|↑↑ HTTP]]**

---

## 1. Date

```http
Date: Tue, 13 May 2026 12:00:00 GMT
```

### 형식
- **RFC 5322 / 7231** — `Day, DD Mon YYYY HH:MM:SS GMT`
- **GMT 강제** — 시간대 없음

### 응답 — 거의 필수
- 캐시 / 304 계산
- 클라 시간 동기

### 요청 — 옵션
- 거의 안 보냄
- 일부 자동 (curl 등)

### 함정
- 시계 동기 (NTP) 안 됨 — 캐시 / 인증 오류
- AWS Signature V4 등 — 시계 5 분 이내

---

## 2. Connection

```http
Connection: keep-alive       (HTTP/1.1 기본)
Connection: close            (응답 후 종료)
Connection: Upgrade          (WebSocket / h2c)
```

### Hop-by-hop
- 다음 hop 만 (Proxy 가 forward X)
- 의도된 hop-by-hop 헤더 목록을 Connection 에 명시

### HTTP/2/3 에서 제거
- HTTP/2 — `connection` 헤더 금지
- 연결 관리는 transport 가 처리

### Keep-Alive 헤더

```http
Connection: keep-alive
Keep-Alive: timeout=5, max=100
```

→ 5 초 후 종료 / 최대 100 요청.

자세히 → [[../performance/keep-alive]]

---

## 3. Cache-Control

요청 + 응답 모두.

```http
Cache-Control: max-age=300, public, must-revalidate
Cache-Control: no-cache, no-store
```

### 응답 — 캐시 정책

| 지시자 | 의미 |
| --- | --- |
| `max-age=N` | N 초 동안 fresh |
| `s-maxage=N` | 공유 캐시 (CDN) max-age |
| `public` | 누구나 캐시 |
| `private` | 사용자 별 (브라우저만) |
| `no-cache` | 매번 origin 검증 (캐시 가능) |
| `no-store` | 캐시 X (민감 정보) |
| `must-revalidate` | 만료되면 반드시 검증 |
| `proxy-revalidate` | proxy 만 |
| `immutable` | 변화 없음 (정적 자원) |
| `stale-while-revalidate=N` | 만료 후 N 초 동안 사용 OK (백그라운드 갱신) |
| `stale-if-error=N` | origin 오류 시 N 초 사용 |

### 요청 — 캐시 동작

```http
Cache-Control: no-cache       (강제 revalidation)
Cache-Control: only-if-cached (캐시 만, 없으면 504)
Cache-Control: max-age=0      (no-cache 와 동등)
```

자세히 → [[../caching/cache-control]]

---

## 4. Pragma (deprecated)

```http
Pragma: no-cache
```

- HTTP/1.0 시절 — `Cache-Control: no-cache` 와 같음
- 모던: Cache-Control 사용
- 옛 IE 호환성 위해 같이 보내는 경우

---

## 5. Via

```http
Via: 1.1 proxy1.example.com (Apache/2.4)
Via: 1.1 proxy1, 1.1 proxy2 (Squid/4.0)
```

### 동작
- 각 Proxy 가 자기 정보 추가
- Proxy chain 추적 (TRACE 의 일부)

### 보안
- 내부 인프라 노출 — 일부 환경에서 제거 권장

---

## 6. Trailer (chunked 의 후행 헤더)

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Trailer: Expires, Content-MD5

5\r\n
hello\r\n
0\r\n
Expires: ...
Content-MD5: ...\r\n
\r\n
```

### 사용
- 헤더 값을 응답 시작 시 모를 때 (스트림 끝의 체크섬 등)
- gRPC over HTTP/2 — Trailer 로 status code

### 한계
- 일부 client / proxy 가 무시
- HTTP/2 / gRPC 외 거의 사용 X

---

## 7. Transfer-Encoding

```http
Transfer-Encoding: chunked
Transfer-Encoding: gzip, chunked
```

### vs Content-Encoding

| 측면 | Transfer-Encoding | Content-Encoding |
| --- | --- | --- |
| 적용 | hop-by-hop | end-to-end |
| 의미 | 전송 방식 | 자원 자체의 인코딩 |
| 예 | chunked | gzip |

→ `Content-Encoding: gzip` — 자원 자체가 압축됨
→ `Transfer-Encoding: gzip` — 전송 시만 압축 (다음 hop 에서 해제)

자세히 → [[../performance/transfer-encoding]]

### HTTP/2/3
- Transfer-Encoding 제거 (frame 이 처리)
- Content-Encoding 만 사용

---

## 8. Upgrade

```http
GET /chat HTTP/1.1
Connection: Upgrade
Upgrade: websocket
```

- 프로토콜 전환 요청
- 응답: `101 Switching Protocols` 또는 무시
- WebSocket / h2c (HTTP/2 cleartext)

---

## 9. Warning (deprecated)

```http
Warning: 110 anderson/1.3.37 "Response is stale"
```

- 캐시 관련 경고
- 모던: 거의 사용 X — RFC 7234 deprecated

---

## 10. Keep-Alive

```http
Keep-Alive: timeout=5, max=100
```

- HTTP/1.1 의 persistent connection 파라미터
- 거의 자동 — 응용 코드는 안 신경

---

## 11. MIME-Version (HTTP 0.9/1.0)

```http
MIME-Version: 1.0
```

- 옛 HTTP/1.0 — 이메일 MIME 와 호환성
- 모던 HTTP 거의 X

---

## 12. 함정

### 함정 1 — Date 시계
NTP 안 되면 캐시 / 인증 깨짐.

### 함정 2 — Pragma + Cache-Control
모순 시 Cache-Control 우선.

### 함정 3 — Connection 의 HTTP/2 위반
HTTP/2 응답에 Connection 헤더 — RFC 위반, 미들박스 거부.

### 함정 4 — Transfer-Encoding / Content-Length 혼동
둘 다 있으면 — **Request Smuggling** 위험. 한쪽만.

### 함정 5 — Via 의 내부 노출
내부 호스트명 / 버전 노출 — Production 에서 제거.

### 함정 6 — Trailer 미지원
HTTP/2 외에는 거의 X. 폴백 필요.

---

## 13. 학습 자료

- **RFC 9110** Section 6, 9
- **RFC 9111** (Caching) — Cache-Control
- MDN — Cache-Control / Connection / Date

---

## 14. 관련

- [[headers]] — Headers hub
- [[../caching/cache-control]] — Cache-Control 깊이
- [[../performance/keep-alive]] — Connection
- [[../performance/transfer-encoding]]
- [[../streaming/chunked-transfer]]
