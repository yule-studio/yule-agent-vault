---
title: "Edge Computing — Workers / Lambda@Edge"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T07:20:00+09:00
tags:
  - network
  - cdn
  - edge
  - workers
---

# Edge Computing — Workers / Lambda@Edge

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Cloudflare Workers / Lambda@Edge / Compute@Edge |

**[[cdn-proxy|↑ CDN/Proxy]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

**CDN edge 에서 코드 실행** — 사용자 가까이에서 JS / WASM 으로 라우팅 / 변환 / 인증 / 개인화.

---

## 2. 왜 Edge?

### Latency
```
Origin (미국):        200 ms RTT
Edge (사용자 근처):    10 ms RTT
```

### 부하
- Origin 보호
- Auto-scale (전 세계)

### 비용
- Egress ↓ (edge 에서 응답)
- 작은 함수 — 매우 저렴

---

## 3. 제품 비교

| 제품 | 런타임 | 시작 시간 | 가격 |
| --- | --- | --- | --- |
| **Cloudflare Workers** | V8 isolate | ~5 ms | 매우 저렴 (free tier) |
| **AWS Lambda@Edge** | Node.js / Python | 100+ ms (cold) | 비쌈 |
| **CloudFront Functions** | JavaScript (V8) | ~1 ms | 저렴 (제한 기능) |
| **Fastly Compute@Edge** | WASM (Rust/Go/JS) | ~1 ms | 보통 |
| **Vercel Edge Functions** | V8 (Cloudflare) | ~5 ms | Vercel 통합 |
| **Deno Deploy** | V8 | ~5 ms | 신규 |
| **Akamai EdgeWorkers** | JavaScript | 빠름 | 엔터프라이즈 |

---

## 4. Cloudflare Workers — V8 Isolate

### 정의
- Chrome 의 V8 — 같은 process 에 수천 isolate
- Container / VM 없음 → cold start ~5 ms

### 코드
```javascript
export default {
    async fetch(request, env, ctx) {
        const url = new URL(request.url);
        
        // A/B testing
        const variant = url.searchParams.get('v') ?? 'A';
        
        // Geographic routing
        const country = request.cf.country;
        if (country === 'CN') {
            return Response.redirect('https://cn.example.com', 302);
        }
        
        // Modify response
        const response = await fetch(request);
        const html = await response.text();
        return new Response(
            html.replace('Hello', 'Hi'),
            response
        );
    }
};
```

### 기능
- KV (key-value)
- Durable Objects (stateful)
- D1 (SQLite)
- R2 (object storage, S3 compat)
- Queue
- AI inference

---

## 5. AWS Lambda@Edge

### 정의
- Lambda 함수를 CloudFront edge 에서
- Node.js / Python

### 4 가지 trigger
```
Viewer Request    — 요청 받음 (CF 시작)
Origin Request    — origin 으로 보내기 전 (cache miss)
Origin Response   — origin 응답 받음
Viewer Response   — 클라이언트 응답 전
```

### 한계
- Cold start (~100 ms)
- 함수 크기 / 시간 제한 (Lambda@Edge — 5s for Origin/Viewer)
- 가격 비쌈

### 모던
- CloudFront Functions — viewer request/response 만, ms 단위, 저렴

---

## 6. Fastly Compute@Edge

### 정의
- WebAssembly (WASM) 기반
- Rust / Go / JavaScript / Swift / TinyGo

### 효과
- 매우 빠름 (~1 ms cold)
- 안전 (sandbox)
- 큰 코드 베이스 가능

### 코드 (Rust)
```rust
use fastly::{Error, Request, Response};

#[fastly::main]
fn main(req: Request) -> Result<Response, Error> {
    let url = req.get_url_str();
    let resp = Response::from_body("Hello from edge!");
    Ok(resp)
}
```

---

## 7. 사용 패턴

### 7.1 A/B Testing
```javascript
const variant = Math.random() < 0.5 ? 'A' : 'B';
request.headers.set('X-Variant', variant);
```

### 7.2 Geolocation
```javascript
const country = request.cf.country;
const region = countryToRegion(country);
request.url = request.url.replace(/^\/api\//, `/api/${region}/`);
```

### 7.3 Auth / JWT
```javascript
const token = request.headers.get('Authorization');
if (!verifyJWT(token)) {
    return new Response('Unauthorized', { status: 401 });
}
```

### 7.4 Image 변환
```
/image.jpg?w=300&q=80
        ↓ edge
이미지 resize / compress / WebP 변환
        ↓ cache
이후 cache hit
```

### 7.5 SSR (Server-Side Rendering)
- Next.js / SvelteKit — Edge runtime
- 페이지를 edge 에서 렌더
- DB / API — 가까운 region

### 7.6 Rate limiting / Bot 차단
- Edge 에서 즉시 차단
- Origin 보호

### 7.7 Routing rewrite
```javascript
if (url.pathname.startsWith('/api/v1/')) {
    url.host = 'old-backend.example.com';
} else if (url.pathname.startsWith('/api/v2/')) {
    url.host = 'new-backend.example.com';
}
```

---

## 8. Stateful Edge — Durable Objects

### 정의
- Cloudflare 의 stateful edge primitive
- 한 객체 = 한 region 의 한 instance
- WebSocket / multiplayer / counter

### 사용
- Chat / multiplayer game
- 분산 lock
- Counter / rate limit
- Leader election

---

## 9. Edge KV / Cache

### KV (Cloudflare / Vercel)
- Eventually consistent (전 세계 60s)
- Key-value
- 사용 — config / feature flag

### Durable Storage
- Strong consistency (한 region)
- WebSocket state

### Cache API
```javascript
const cache = caches.default;
let response = await cache.match(request);
if (!response) {
    response = await fetch(request);
    ctx.waitUntil(cache.put(request, response.clone()));
}
return response;
```

---

## 10. WebAssembly System Interface (WASI)

### 정의
- WASM 의 system call 표준
- 모든 WASM edge — 같은 API

### 효과
- Rust / Go / C# / Swift 모두 edge 에서
- 큰 코드베이스 portable

### 도구
- Fermyon Spin
- Wasmtime
- WasmCloud

---

## 11. 한계 / 함정

### 함정 1 — CPU / 시간 제한
Workers — 50 ms (free), 30 s (paid).
Lambda@Edge — Origin 5s, Viewer 5s.
대용량 — 부적합.

### 함정 2 — Memory 제한
Workers — 128 MB. Big data 처리 X.

### 함정 3 — DB 연결
DB connection pool — edge 마다 다름 → connection 폭주.
HTTP-based DB (PlanetScale, Neon) / D1 / 외부 connection pooler (Hyperdrive).

### 함정 4 — Cold start (Lambda)
Lambda@Edge — 100+ ms. Workers / Compute@Edge 권장.

### 함정 5 — Eventually consistent
KV — 60s 지연. Strong needed — Durable Objects.

### 함정 6 — Debugging 어려움
원격 / 분산 — 로그 / trace 가 어려움. Wrangler tail / observability.

### 함정 7 — Lock-in
플랫폼 별 API 다름. WASI 가 표준화 시도.

---

## 12. 학습 자료

- Cloudflare Workers docs
- AWS Lambda@Edge docs
- Fastly Compute@Edge docs
- "Serverless Edge" (Brian LeRoux)
- Vercel Edge Functions docs

---

## 13. 관련

- [[cdn]] — CDN 의 부모
- [[cdn-proxy]] — Hub
- [[reverse-proxy]] — Edge 도 reverse proxy
- [[../topics/serverless]] (TBD)
