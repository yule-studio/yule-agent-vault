---
title: "5xx — Server Errors"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:30:00+09:00
tags:
  - network
  - http
  - status-codes
  - 5xx
---

# 5xx — Server Errors

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 500/501/502/503/504/505/507/508/511 |

**[[status-codes|↑ Status Codes]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

"서버 오류" — 클라 잘못 X. 서버 / 인프라 / 의존성 문제. 클라가 (보통) 재시도.

---

## 2. 표준 코드

### 500 Internal Server Error

```http
HTTP/1.1 500 Internal Server Error
Content-Type: application/json
{"error":"Database connection failed"}
```

#### 의미
- 서버 측 예상 못한 오류 (NullPointerException, 코드 버그 등)
- 가장 일반적 / 모호 — 명확한 5xx 있으면 그것
- 응답 body 에 stack trace 노출 X (보안)

#### 흔한 원인
- 코드 버그 (예외 unhandled)
- DB / 캐시 연결 실패
- 외부 API 실패
- 디스크 / 메모리 부족

#### 모니터링
- 500 = 알림 / 페이저 / 즉시 디버깅
- 일정 비율 (`5xx rate > 0.1%`) 알람

### 501 Not Implemented

```http
HTTP/1.1 501 Not Implemented
```

#### 의미
- 서버가 해당 메서드 / 기능 미구현
- 모든 클라에 대해 (= 405 의 "이 자원 X" 와 다름)
- 영구 — 캐시 가능 (heuristic)

#### 예
- PROPFIND (WebDAV) 가 일반 HTTP 서버에 → 501
- HTTP 의 미래 기능 사용 시

### 502 Bad Gateway

```http
HTTP/1.1 502 Bad Gateway
```

#### 의미
- **Gateway / Proxy / LB** 가 백엔드에서 유효한 응답 못 받음
- 백엔드 크래시 / 잘못된 응답 / 연결 거부

#### 흔한 원인
- Nginx 의 백엔드 (uWSGI / Express) crash / 응답 없음
- AWS ALB / ELB 의 target 비정상
- CDN ↔ Origin 의 오류

#### 디버깅
- 백엔드 로그 확인
- LB / Proxy 의 access log
- Health check 상태

### 503 Service Unavailable

```http
HTTP/1.1 503 Service Unavailable
Retry-After: 60
```

#### 의미
- 일시적 사용 불가
- 점검 / 과부하 / Rate Limit (전체)
- **Retry-After 헤더 권장**

#### 503 vs 429
- 429 — 특정 클라가 너무 많이 (per-client rate limit)
- 503 — 전체 서버 한계 / 점검

#### 활용
- Maintenance 페이지
- Circuit breaker (의존성 죽으면 503)
- Graceful degradation

### 504 Gateway Timeout

```http
HTTP/1.1 504 Gateway Timeout
```

#### 의미
- Gateway / Proxy 가 백엔드 응답 기다리다 timeout
- 502 와 비슷 — 502 는 "잘못된 응답", 504 는 "응답 X"

#### 흔한 원인
- 백엔드 응답 매우 느림 (DB 락, 무한 루프)
- LB timeout (60s 기본) 보다 길어짐
- Nginx `proxy_read_timeout`

#### 해결
- 백엔드 최적화 (DB 인덱스 / 쿼리)
- Timeout 늘림 (마지막 수단)
- 비동기 처리 + 202 Accepted + 폴링

### 505 HTTP Version Not Supported

```http
HTTP/1.1 505 HTTP Version Not Supported
```

- HTTP/0.9 요청 등 — 서버가 거부
- 옛 / 임베디드 클라이언트 문제

### 506 Variant Also Negotiates (RFC 2295)
복잡한 content negotiation 오류. 거의 X.

### 507 Insufficient Storage (WebDAV)

```http
HTTP/1.1 507 Insufficient Storage
```

- 저장 공간 부족 (DB / 디스크 가득)

### 508 Loop Detected (WebDAV)
순환 참조 감지.

### 510 Not Extended
RFC 2774 — Extension Framework. 거의 X.

### 511 Network Authentication Required

```http
HTTP/1.1 511 Network Authentication Required
Content-Type: text/html

<html>...로그인 페이지로 redirect...</html>
```

- **Captive Portal** — 카페 / 공항 Wi-Fi 의 "로그인 페이지"
- HTTPS 사이트가 강제 redirect 되는 신호
- 모던 OS (iOS / macOS / Windows) 가 자동 감지 (`/generate_204` 등)

---

## 3. Cloudflare 5xx (520-526)

Cloudflare 만 사용 (RFC 표준 X):

| 코드 | 의미 |
| --- | --- |
| **520** | Web Server Returned Unknown Error |
| **521** | Origin Server Down |
| **522** | Origin Connection Timed Out |
| **523** | Origin Is Unreachable |
| **524** | Origin Connection Timeout (full HTTP) |
| **525** | SSL Handshake Failed |
| **526** | Invalid SSL Certificate |

→ Cloudflare 사용 시 디버깅. 다른 CDN 은 502/504 로 표준화.

---

## 4. 재시도 정책

### 클라이언트
- **500 / 502 / 503 / 504** — 재시도 OK (Exponential backoff)
- **501 / 505** — 영구 — 재시도 X

### 라이브러리 기본
```
requests (Python) + Retry: 500, 502, 503, 504
fetch (JS) 기본: 재시도 X — 직접 구현
HTTPClient (Go): 직접 구현
```

### Backoff 전략

```
1s → 2s → 4s → 8s → 16s + Jitter (random ms)
최대 5 회
```

### Circuit Breaker

```
실패율 임계치 (50%) 초과 → 일정 시간 즉시 503
→ Recovery 시도
```

자세히 → [[../../topics/topics]]

---

## 5. 응답 body 의 보안

### 안전
```json
{
  "error": "INTERNAL_ERROR",
  "code": "E5001",
  "request_id": "abc-123"
}
```

→ 클라가 request_id 로 지원 요청, 서버는 로그에서 추적.

### 위험 — 절대 X
```json
{
  "error": "Database connection failed: postgres://user:pass@db.internal:5432/...",
  "stack_trace": "java.lang.NullPointerException at..."
}
```

→ 인프라 / 자격 / 코드 구조 노출.

---

## 6. Maintenance Mode

### 503 + Retry-After
```http
HTTP/1.1 503 Service Unavailable
Content-Type: text/html
Retry-After: 1800        (30 분)
Cache-Control: no-cache

<html><body>점검 중. 30 분 후 재시도.</body></html>
```

→ 검색엔진 색인 보호 (200 + "점검 중" 페이지 X — 색인 잘못 갱신).

### 로드 밸런서 기능
- AWS ALB — Listener rule 로 503 응답
- Nginx — `return 503;` 또는 maintenance HTML

---

## 7. SRE — Error Budget

```
SLO: 99.9% 성공
Error Budget: 0.1% (43 분 / 월)

5xx rate 가 0.1% 초과 → 페이저 / 신규 deploy 중단
```

자세히 → [[../../../distributed-systems/distributed-systems]]

---

## 8. 모니터링 / 알람

### 핵심 SLI (Service Level Indicator)
- **Availability** = (2xx + 3xx) / Total
- **Error rate** = 5xx / Total
- **Latency** — p50, p95, p99

### Prometheus 알람
```yaml
- alert: HighErrorRate
  expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.01
  for: 5m
  annotations:
    summary: "5xx error rate > 1%"
```

---

## 9. 함정

### 함정 1 — Stack trace 노출
보안 위험. Production 은 일반 메시지만.

### 함정 2 — 모든 오류를 500
인프라 (502/504) vs 응용 (500) 구분.

### 함정 3 — 재시도 폭주
지수 백오프 + Jitter 없으면 thundering herd.

### 함정 4 — 502 / 503 / 504 혼동
- 502 — 백엔드의 잘못된 응답
- 503 — 의도적 점검 / 과부하
- 504 — 백엔드 응답 X (timeout)

### 함정 5 — Health check 가 200 만 보고
일부 백엔드가 죽었는데 LB 가 모름. Deep health check.

### 함정 6 — Long timeout 의 함정
60s+ timeout → 사용자 경험 ↓. 비동기 처리.

### 함정 7 — 503 의 검색엔진 영향
짧으면 OK. 며칠 지속 → 색인 영향. 점검 시 명시.

---

## 10. 학습 자료

- **RFC 9110** Section 15.6
- Google SRE Book — Error Budget
- Cloudflare error codes — https://developers.cloudflare.com/support/troubleshooting/
- "Site Reliability Engineering" — Beyer / Jones / Petoff / Murphy

---

## 11. 관련

- [[status-codes]] — hub
- [[4xx-client-errors]] — 클라 vs 서버
- [[../headers/response-headers]] — Retry-After
- [[../../topics/topics]] — Circuit Breaker
- [[../../load-balancing/load-balancing]] — LB 의 502/503/504
