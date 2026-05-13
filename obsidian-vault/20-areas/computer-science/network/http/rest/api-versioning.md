---
title: "API Versioning"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:55:00+09:00
tags:
  - network
  - http
  - rest
  - versioning
---

# API Versioning

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | URL / Header / Media type / Deprecation |

**[[rest|↑ REST]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 왜 버저닝

```
초기:  POST /users {name: "Alice"}
              ↓
변경:  POST /users {first_name: "Alice", last_name: "Doe"}

기존 클라이언트:
  request body 그대로 → 깨짐
```

→ Breaking change 시 옛 클라 지원 + 새 기능 도입.

---

## 2. 4 가지 방식

### 2.1 URL Versioning (path)

```
/api/v1/users
/api/v2/users
```

#### 장점
- 가장 명시적
- 디버깅 쉬움
- curl / 브라우저 OK

#### 단점
- "URI 는 자원만" — Fielding 의 비판
- 같은 자원이 다른 URL

#### 사용
- 가장 흔함 (Stripe / GitHub / Twilio 등)

### 2.2 Header Versioning

```http
GET /users HTTP/1.1
Accept: application/vnd.example.v2+json
```

또는:

```http
GET /users HTTP/1.1
API-Version: 2
X-API-Version: 2
```

#### 장점
- URI 가 자원만 유지
- RESTful 한 시각

#### 단점
- 디버깅 어려움 (헤더 없으면)
- 브라우저 직접 사용 X
- 캐시 키 복잡

### 2.3 Query Parameter

```
/users?version=2
/users?v=2
```

#### 장점
- 단순 — 기존 URL 에 옵션
- 직접 사용 OK

#### 단점
- 더러움
- 기본 버전 모호

### 2.4 Subdomain

```
v1.api.example.com
v2.api.example.com
```

#### 사용
- 인프라 완전 분리 (다른 서버)
- 큰 마이그레이션

---

## 3. 비교 표

| 방식 | URL | Header | Query | Subdomain |
| --- | --- | --- | --- | --- |
| 명시성 | 높음 | 낮음 | 중 | 높음 |
| RESTful | 약 | 강 | 중 | 약 |
| 디버깅 | 쉬움 | 어려움 | 쉬움 | 쉬움 |
| 인프라 | 같음 | 같음 | 같음 | 분리 |
| 캐시 | 별도 키 | Vary 필요 | 별도 키 | 별도 |
| 사용 | 보편 | 학술적 | 일부 | 큰 변경 |

---

## 4. Semantic Versioning?

```
v1.2.3
 │ │ └ Patch (버그 수정)
 │ └─ Minor (호환 + 새 기능)
 └── Major (호환 깨짐)
```

API 는 보통 **Major 만** 노출:
```
/api/v1   ← v1.x.y 의 모든 minor/patch
/api/v2
```

- Minor / Patch 는 별도 표시 X (또는 `X-API-Version-Build` 등 메타)
- 응용 / 클라가 알 필요 X (호환 보장)

---

## 5. Breaking vs Non-breaking Change

### Non-breaking (호환)
- 새 필드 추가 (옵션)
- 새 endpoint 추가
- 새 응답 코드 (5xx 의 일부)
- 더 관대한 입력 (string + int 둘 다)

→ 같은 버전 유지.

### Breaking (호환 깨짐)
- 필드 제거
- 필드 의미 변경
- 응답 구조 변경
- 필수 입력 추가
- URI 변경
- 메서드 변경

→ 새 버전.

### 한계
- 일부 변경의 호환성 모호 (예: enum 추가 — 클라가 모름 → 처리 못 함?)
- 정책 — "신중하게 + 통보"

---

## 6. Deprecation 정책

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Wed, 31 Dec 2026 23:59:59 GMT
Link: <https://example.com/api/v2/users>; rel="successor-version"
Warning: 299 - "v1 deprecated. Use /api/v2/users."
```

### RFC 8594 (Sunset 헤더)
- `Sunset` — 종료 예정 시점
- 통보 기간 (6 개월 - 1 년)

### Deprecation 헤더 (RFC 9745)
- `Deprecation: true` 또는 timestamp

### 단계
1. **Announce** — 변경 예고 (블로그 / changelog)
2. **Deprecate** — 헤더 추가 + 문서
3. **Warn** — 로그 / 응답에 경고
4. **Sunset** — 명시 시점에 종료 (410 Gone 또는 redirect)

---

## 7. Monkey Patch 방식 — 매우 안 좋음

```
POST /users HTTP/1.1
X-Force-Old-Behavior: true     ← 절대 X
```

→ 비표준 + 일관성 깨짐.

---

## 8. Implicit Versioning

```
계약 변경 없이 점진 발전:
- v1 만 — 모든 클라 호환

방식:
- Additive only (필드 추가 / 새 endpoint)
- Strict input validation (모르는 필드 무시 — Postel's Law)
- Default 값 명시
```

### Postel's Law
> "Be conservative in what you do, be liberal in what you accept from others"

- 응답은 엄격 (변경 X)
- 요청은 관대 (모르는 필드 무시)

---

## 9. 실제 패턴 — 회사 별

### Stripe
- URL 의 date 기반 versioning: `2024-04-10`
- `Stripe-Version` 헤더 또는 dashboard 설정
- Backward compatibility 매우 길게

### GitHub
- `Accept: application/vnd.github+json; version=2022-11-28`
- v3 (REST) + v4 (GraphQL)

### Twilio
- URL `/2010-04-01/...`
- 거의 변경 X (오래된 버전 유지)

### AWS
- 매우 보수적 — 거의 같은 API 수년
- 새 기능은 새 서비스 / 새 API

---

## 10. 함정

### 함정 1 — 잦은 Major version
v2, v3, v4... 빠른 deprecation — 클라 부담.

### 함정 2 — 옛 버전 무한 유지
유지 비용 ↑ — Sunset 정책.

### 함정 3 — Deprecation 통보 없이 종료
클라 갑자기 깨짐. 6-12 개월 통보.

### 함정 4 — Header / Query / URL 혼합
일관 — 한 가지만.

### 함정 5 — Minor 의 의도치 않은 breaking
"추가" 라도 응답 구조 변화 → 옛 클라 깨짐. Strict additive.

### 함정 6 — 응용 코드의 버전 분기
```python
if version == "v1":
    ...
elif version == "v2":
    ...
```
→ 복잡. Adapter / Translation layer.

---

## 11. 학습 자료

- **RFC 8594** (Sunset)
- **RFC 9745** (Deprecation 헤더)
- "API Versioning" — Stripe blog
- "Versioning REST APIs" — Microsoft REST API Guidelines

---

## 12. 관련

- [[rest]] — REST hub
- [[resource-design]] — URI
- [[../headers/response-headers]] — Sunset, Deprecation
