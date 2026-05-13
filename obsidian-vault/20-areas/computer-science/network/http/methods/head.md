---
title: "HEAD — 헤더만 조회"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:55:00+09:00
tags:
  - network
  - http
  - head
---

# HEAD — 헤더만 조회

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 헤더 메타 조회 / 캐시 확인 / Body X |

**[[methods|↑ Methods]]** · **[[../http|↑↑ HTTP]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **안전** | ✅ |
| **멱등** | ✅ |
| **캐시 가능** | ✅ |
| **Body** | ❌ (응답에도 body 없음) |
| **도입** | HTTP/1.0 |

---

## 1. 한 줄 정의

**GET 과 동일하지만 응답 body 만 생략**. 헤더 (Content-Length / Last-Modified /
ETag) 만 받음.

---

## 2. 요청 / 응답

```http
HEAD /large-file.zip HTTP/1.1
Host: example.com

```

```http
HTTP/1.1 200 OK
Content-Type: application/zip
Content-Length: 1073741824
Last-Modified: Mon, 13 May 2026 10:00:00 GMT
ETag: "abc123"
Accept-Ranges: bytes

(body 없음)
```

---

## 3. HEAD 가 답하는 질문

### 3.1 자원이 존재? (200 vs 404)
```bash
curl -I https://example.com/page    # -I = HEAD
HTTP/1.1 200 OK         ← 있음
또는
HTTP/1.1 404 Not Found  ← 없음
```

### 3.2 파일 크기?
```http
Content-Length: 1073741824   ← 1 GB
```

→ 큰 파일 다운로드 전 크기 확인.

### 3.3 마지막 변경 시간?
```http
Last-Modified: ...
ETag: "..."
```

→ 캐시 valid 확인 (조건부 GET 전).

### 3.4 Content-Type?
```http
Content-Type: image/jpeg
```

→ MIME 확인.

### 3.5 Range 지원?
```http
Accept-Ranges: bytes
```

→ 부분 다운로드 가능?

---

## 4. 사용 사례

### 4.1 다운로드 매니저
- 큰 파일 크기 확인
- Range 지원 확인
- ETag 로 변경 감지

### 4.2 캐시 검증
```
캐시: ETag "v1" 보유
HEAD /resource → ETag "v2"   ← 변경됨
→ GET 으로 새 본문 가져옴
```

### 4.3 Link Checker
- 모든 링크의 200/404 확인
- Body 안 다운로드 → 빠름 / 대역폭 절약

### 4.4 Health Check
```
HEAD /health HTTP/1.1
→ 200 OK   ← 살아있음
```

### 4.5 Pre-flight Check
- 큰 업로드 전 서버 의사 확인
- Expect: 100-continue 대안

---

## 5. GET 과의 차이

| 측면 | GET | HEAD |
| --- | --- | --- |
| 요청 | 본문 X | 본문 X |
| 응답 헤더 | 동일 | 동일 |
| 응답 본문 | ✅ | ❌ |
| 대역폭 | 큼 | 작음 |
| 서버 부하 | 보통 | 작음 (body 안 생성) |
| HTTP/2 frame | DATA | HEADERS 만 |

→ **요청은 같고, 응답은 body 만 다름**.

---

## 6. 서버의 구현

이상적:
```python
def head(self):
    # 헤더만 계산
    headers = self.compute_headers()
    return Response(status=200, headers=headers, body=None)
```

실제로 일부 서버:
```python
def head(self):
    # GET 처리 후 body 버림 (비효율)
    response = self.get()
    response.body = None
    return response
```

→ 모던 framework 는 분리.

---

## 7. 응답 코드

| 코드 | 의미 |
| --- | --- |
| **200 OK** | 자원 있음 (헤더만) |
| **301 / 302** | Redirect (Location 헤더만 의미) |
| **304 Not Modified** | 캐시 valid |
| **404 Not Found** | 없음 |
| **405 Method Not Allowed** | HEAD 미지원 (드뭎) |

---

## 8. curl 예

```bash
# -I 또는 --head
curl -I https://example.com/file.zip

# 출력:
# HTTP/1.1 200 OK
# Content-Type: application/zip
# Content-Length: 1073741824
# Last-Modified: ...

# 헤더 + verbose
curl -I -v https://example.com/

# 조건부 HEAD (캐시 검증)
curl -I -H 'If-None-Match: "v1"' https://example.com/
# → 304 Not Modified 또는 200 OK + 새 ETag
```

---

## 9. fetch (JS)

```javascript
const r = await fetch("/file.zip", {method: "HEAD"});
const size = r.headers.get("Content-Length");
const lastMod = r.headers.get("Last-Modified");
console.log(`${size} bytes, modified ${lastMod}`);
```

---

## 10. Python — requests

```python
import requests
r = requests.head("https://example.com/file.zip", allow_redirects=True)
print(r.headers["Content-Length"])
```

---

## 11. HEAD vs GET 의 미세 차이

### 11.1 일부 서버 / framework
- Content-Length 가 HEAD 응답에 없을 수 있음 (chunked 등)
- HEAD 가 GET 보다 다른 캐시 정책

### 11.2 Proxy / CDN
- HEAD 응답을 GET 처럼 캐시? 별도?
- Vary 헤더로 분리

### 11.3 RFC 권장
- HEAD 응답의 헤더 = GET 응답의 헤더 (body 만 X)
- 일치 안 하면 RFC 위반

---

## 12. 함정

### 함정 1 — HEAD 미지원 서버
일부 서버 / API 가 HEAD 비활성 → 405 Method Not Allowed. GET 대체.

### 함정 2 — HEAD 응답에 body
RFC 위반 — 일부 옛 서버 / proxy 가 보냄. 클라가 무시.

### 함정 3 — HEAD 와 GET 헤더 불일치
캐시 일관성 깨짐. 서버가 동일 헤더 계산.

### 함정 4 — Content-Length 와 실제 차이
HEAD 가 잘못된 길이 알림 → 다운로드 매니저 문제.

### 함정 5 — Pre-fetch / 봇이 GET 대신 HEAD?
일부 — 안전한 메서드라 자유롭게. 분석 도구가 HEAD 도 포함해야.

---

## 13. 학습 자료

- **RFC 9110** Section 9.3.2
- MDN HEAD
- curl `-I` 가이드

---

## 14. 관련

- [[methods]] — Methods hub
- [[get]] — 본문 포함 버전
- [[options]] — 메서드 조회와 비교
- [[../caching/etag-conditional]]
