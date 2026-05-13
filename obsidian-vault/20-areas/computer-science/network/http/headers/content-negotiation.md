---
title: "Content Negotiation — Accept-* / Vary"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:00:00+09:00
tags:
  - network
  - http
  - headers
  - content-negotiation
---

# Content Negotiation — Accept-* / Vary

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Accept / Accept-Language / Accept-Encoding / Vary / q-values |

**[[headers|↑ Headers]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

같은 자원의 **여러 표현** 중 클라-서버 협상으로 최적 선택. MIME / 언어 / 인코딩 /
문자셋 4 차원.

---

## 2. 동작

```
Client → Accept: application/json
       → Accept-Language: ko, en;q=0.5
       → Accept-Encoding: gzip, br

Server → 가용 표현 중 가장 좋은 매칭
       → Content-Type: application/json; charset=UTF-8
       → Content-Language: ko
       → Content-Encoding: gzip
       → Vary: Accept-Encoding, Accept-Language    ← 캐시 키
```

---

## 3. Accept — MIME 협상

```http
Accept: application/json
Accept: application/json, text/html;q=0.9, */*;q=0.5
Accept: image/avif, image/webp, image/*, */*;q=0.8
```

### q-value (Quality)

- 0.0 - 1.0 (기본 1.0)
- 큰 값 = 선호
- `*/*` = 모든 것 (최후 선택)

### 예 — 모던 브라우저의 이미지 요청

```http
Accept: image/avif, image/webp, image/apng, image/svg+xml, image/*, */*;q=0.8
```

→ 서버는 가능하면 AVIF, 안 되면 WebP, 안 되면 일반 이미지.

### 협상 알고리즘

```
1. 클라 Accept 의 각 type 점수 (q × 가용성)
2. 가장 높은 점수의 표현 선택
3. 가용 매칭 없으면 → 406 Not Acceptable 또는 기본 표현
```

---

## 4. Accept-Language — 언어 협상

```http
Accept-Language: ko-KR, ko;q=0.9, en-US;q=0.8, en;q=0.7
```

### 형식
- BCP 47 (RFC 5646)
- 우선순위: 정확한 매칭 → 언어만 → fallback

### 매칭

```
Accept-Language: ko
가용: ko, en
→ ko 선택

Accept-Language: ko-KR
가용: ko (no ko-KR)
→ ko 선택 (prefix match)

Accept-Language: en-AU
가용: en-US, en-GB
→ en-US 또는 en-GB 중 하나 (정책)
```

### 함정
- 사용자 OS 의 시스템 언어 — 의도 X 일 수 있음
- 사용자 명시 선택 (cookie / param) 우선
- URL `/ko/...` 으로 명시도 흔함

---

## 5. Accept-Encoding — 압축 협상

```http
Accept-Encoding: gzip, deflate, br
Accept-Encoding: gzip;q=1.0, br;q=0.9
Accept-Encoding: *
Accept-Encoding:                       (빈 값 = identity 만)
```

### 응답

```http
Content-Encoding: br
```

### 우선순위 (서버)
- Brotli (br) — 가장 작음, 모든 모던 브라우저
- gzip — 보편적
- zstd — 모던 (Cloudflare 등)
- deflate — 사용 X

자세히 → [[../performance/compression-encoding]]

---

## 6. Accept-Charset (deprecated)

```http
Accept-Charset: utf-8, iso-8859-1;q=0.5
```

- HTTP/1.1 도입, 거의 사용 X
- 모던: UTF-8 표준 — 협상 불필요
- RFC 9110 — "사용 권장 X"

---

## 7. Vary — 캐시 키

```http
Vary: Accept-Encoding, Accept-Language
```

### 의미
- 응답이 어느 요청 헤더에 따라 다른지
- 캐시가 키에 포함

### 캐시 키 예
```
URL + Method + Vary 헤더들
/api/users + GET + (Accept-Encoding: gzip) → 캐시1
/api/users + GET + (Accept-Encoding: br)   → 캐시2
```

### Vary 의 함정

#### `Vary: *`
- 모든 요청 헤더에 따라 다름
- **사실상 캐시 불가**

#### `Vary: User-Agent`
- UA 가 거의 무한 변형
- 캐시 효율 폭락
- → 차라리 응용 / 라우팅에서 분기

#### `Vary: Cookie`
- 사용자별 응답
- 공유 캐시 (CDN) 무효 — `Cache-Control: private`

자세히 → [[../caching/vary-header]]

---

## 8. URL 기반 vs Negotiation

### 8.1 Negotiation (협상)

```
GET /api/users/123
Accept: application/json
→ JSON

GET /api/users/123
Accept: application/xml
→ XML
```

- 같은 URI — 표현이 협상
- RESTful — URI 가 자원만

### 8.2 URL 기반 (Suffix / Param)

```
GET /api/users/123.json
GET /api/users/123.xml
GET /api/users/123?format=json
```

- URI 가 표현 결정
- 단순 / 명시적
- 일부 응용 (Stripe Webhook, GitHub API)

### 8.3 차이

| 측면 | Negotiation | URL 기반 |
| --- | --- | --- |
| URI | 자원만 | 표현 포함 |
| 캐시 키 | Vary 필요 | URI 자체 |
| 디버깅 | 헤더 봐야 | URL 보면 됨 |
| 사용자 친화 | 약 | 강 |

---

## 9. 응용 예 — Web API

### JSON / XML 둘 다 지원

```python
# FastAPI
@app.get("/users/{user_id}")
def get_user(user_id: int, accept: str = Header(None)):
    user = ...
    if "application/xml" in accept:
        return Response(content=to_xml(user), media_type="application/xml")
    return user    # JSON 기본
```

### 다국어

```python
@app.get("/")
def home(accept_language: str = Header("en")):
    locale = parse_accept_language(accept_language)
    return render("home", locale=locale)
```

### 이미지 포맷 (Picture)

```html
<picture>
  <source srcset="image.avif" type="image/avif">
  <source srcset="image.webp" type="image/webp">
  <img src="image.jpg" alt="...">
</picture>
```

또는 서버 측 — UA / Accept 보고 동적 변환 (Cloudinary, Image CDN).

---

## 10. 함정

### 함정 1 — Accept 의 미세 점수
실제 q 차이는 미미. 명확한 우선순위 정의 어려움.

### 함정 2 — Accept-Charset 사용
deprecated. UTF-8.

### 함정 3 — Vary 누락
캐시가 잘못된 인코딩 / 언어 반환.

### 함정 4 — Vary 의 효율
`Vary: User-Agent` 같은 광범위한 Vary = 캐시 무효.

### 함정 5 — Negotiation 의 디버깅 어려움
같은 URL 이 다른 응답. cURL 시 명시: `-H "Accept: application/json"`.

### 함정 6 — 옛 클라의 q 무시
일부 옛 라이브러리가 q 무시 → 잘못된 표현.

### 함정 7 — 라이브러리의 default Accept
fetch / requests 의 기본 `*/*` → 서버가 임의 선택. 명시 권장.

---

## 11. 학습 자료

- **RFC 9110** Section 12 (Content Negotiation)
- **RFC 7231 Section 5.3** (옛)
- MDN Content Negotiation
- "Negotiation in HTTP" (W3C)

---

## 12. 관련

- [[headers]] — Headers hub
- [[request-headers]] — Accept-* 헤더 위치
- [[response-headers]] — Vary, Content-*
- [[../caching/vary-header]] — Vary 깊이
- [[../performance/compression-encoding]] — Accept-Encoding
