---
title: "Reverse Proxy"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T07:15:00+09:00
tags:
  - network
  - proxy
  - reverse-proxy
  - nginx
---

# Reverse Proxy

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Nginx / TLS / 캐시 / 라우팅 |

**[[cdn-proxy|↑ CDN/Proxy]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

**서버 측 중간자** — 클라이언트는 reverse proxy 만 알고 backend 는 모름. 라우팅 / 부하 분산 / TLS / 캐시 / 보안.

---

## 2. 동작

```
Client → Reverse Proxy: GET / HTTP/1.1
                        Host: example.com

Reverse Proxy:
  - Rule 매칭
  - Backend 선택
  - 헤더 변환

Reverse Proxy → Backend: GET / HTTP/1.1
                         Host: backend
                         X-Forwarded-For: client_ip

Backend → Reverse Proxy: 200 OK
Reverse Proxy → Client: 200 OK (헤더 수정)
```

---

## 3. 책임

| 책임 | 설명 |
| --- | --- |
| **Load balancing** | 여러 backend 분산 |
| **TLS termination** | HTTPS 종료 |
| **Caching** | 자주 요청되는 응답 |
| **압축** | gzip / br |
| **인증** | Basic / OAuth / JWT |
| **Rate limit** | per IP / per user |
| **WAF** | OWASP 필터 |
| **Header 변환** | X-Forwarded-For, X-Real-IP |
| **버전 변환** | HTTP/2 ↔ HTTP/1.1 |
| **압축 / 변환** | image / minify |

---

## 4. Nginx

### 가장 흔한
```nginx
http {
    upstream backend {
        server 10.0.0.1:8080;
        server 10.0.0.2:8080;
    }
    
    server {
        listen 443 ssl http2;
        server_name example.com;
        
        ssl_certificate cert.pem;
        ssl_certificate_key key.pem;
        
        gzip on;
        gzip_types text/css application/javascript application/json;
        
        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            
            proxy_buffering on;
            proxy_cache mycache;
        }
        
        location /static/ {
            root /var/www;
            expires 1y;
        }
        
        location /api/ws {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
}
```

### 캐시
```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mycache:10m max_size=1g;

location / {
    proxy_cache mycache;
    proxy_cache_valid 200 1h;
    proxy_cache_key "$scheme$request_method$host$request_uri";
    add_header X-Cache-Status $upstream_cache_status;
}
```

---

## 5. HAProxy / Envoy / Traefik

자세히 → [[../load-balancing/l7-load-balancing]]

### 비교
| | 강점 |
| --- | --- |
| **Nginx** | 안정 / 정적 파일 / 캐시 |
| **HAProxy** | 매우 빠름 / L4+L7 |
| **Envoy** | 동적 / Service Mesh |
| **Traefik** | K8s / auto-discovery |
| **Caddy** | Auto HTTPS (Let's Encrypt) |

---

## 6. TLS Termination

### 모드
```
Client ──TLS──→ Reverse Proxy ──평문──→ Backend     (offload)
Client ──TLS──→ Reverse Proxy ──TLS───→ Backend     (re-encrypt)
Client ──TLS──→ Reverse Proxy ──TLS───→ Backend     (passthrough, L4 only)
```

### Offload (가장 흔함)
- Reverse proxy 가 cert 관리
- Backend 평문 — 내부 망 가정
- 부하 적음

### Re-encrypt
- E2E + 검사
- 부하 큼

### Passthrough
- L4 LB — TLS handshake 통과
- Backend 가 cert
- SNI 기반 라우팅만 가능

---

## 7. HTTP 버전 변환

```
Client (HTTP/3) → Reverse Proxy → Backend (HTTP/1.1)
```

### 효과
- 모던 클라이언트 — HTTP/3 의 빠른 connection
- Backend — 단순 HTTP/1.1 유지

### Multiplexing
- HTTP/2 의 stream → HTTP/1.1 의 여러 connection
- 반대 — keepalive

---

## 8. 캐시 — Reverse Proxy 의 강점

### Varnish — 캐시 전용
```vcl
backend default {
    .host = "backend";
    .port = "8080";
}

sub vcl_recv {
    if (req.url ~ "^/api/") {
        return (pass);    # 캐시 X
    }
    return (hash);
}

sub vcl_backend_response {
    set beresp.ttl = 1h;
}
```

### 효과
- Memory-based — 매우 빠름
- ESI 지원
- 변환 (gzip / image)

### Nginx 와 비교
- Varnish — 캐시 전문 (느린 디스크 X)
- Nginx — 다목적

---

## 9. Header 변환

### Request
```
X-Forwarded-For: <client_ip>
X-Real-IP: <client_ip>
X-Forwarded-Proto: https
X-Forwarded-Host: example.com
Forwarded: for=1.2.3.4;proto=https;host=example.com   (RFC 7239)
```

### Response
```
Server: nginx        ← 숨김 권장 (보안)
X-Powered-By: ...    ← 제거
```

---

## 10. WebSocket / gRPC / SSE

### WebSocket
```nginx
location /ws {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 86400;    # long-lived
}
```

### gRPC
```nginx
location / {
    grpc_pass grpc://backend;
}
```

→ Nginx 1.13.10+ — gRPC native.

### SSE (Server-Sent Events)
- buffering 끄기 (proxy_buffering off)
- 장기 connection

---

## 11. Auth / OIDC

### oauth2-proxy
```
Client → Nginx → oauth2-proxy → Google OAuth
                ↓ 인증 OK
                Backend
```

- 모든 요청 — auth 통과
- Backend 는 인증 없이 / 헤더만 신뢰

### Cloudflare Access
- 동일 — SSO + 정책

---

## 12. Rate Limiting

### Nginx
```nginx
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

location / {
    limit_req zone=mylimit burst=20 nodelay;
    proxy_pass http://backend;
}
```

### Tier
- Per IP / per user / per tenant
- 다른 endpoint — 다른 limit (login 은 strict)

---

## 13. Reverse Proxy vs API Gateway

### Reverse Proxy
- 라우팅 / TLS / 캐시 / 로드 분산
- HTTP 위주

### API Gateway
- + 인증 / Rate limit / Quota / API Key
- + 변환 (SOAP → REST)
- + Aggregation
- + Observability / Analytics
- 제품 — Kong, Tyk, AWS API Gateway, Apigee

→ API Gateway 가 Reverse Proxy 의 superset.

---

## 14. 함정

### 함정 1 — Buffering 의 latency
큰 응답 — full buffer 까지 backend 대기. proxy_buffering off (스트리밍).

### 함정 2 — Upstream timeout
proxy_read_timeout 짧음 → long-running 끊김. WebSocket / SSE 별도.

### 함정 3 — X-Forwarded-For 의 신뢰
사용자가 헤더 직접 보냄 — 신뢰 가능한 proxy 까지만 trust.

### 함정 4 — Cache 의 Vary
Accept-Encoding 안 넣으면 — gzip 버전이 평문 사용자에게.

### 함정 5 — TLS 종료 후 backend 평문
내부 망 — 안전 가정. mTLS / private VPC.

### 함정 6 — 한 Backend 죽으면 502
Active health check / fallback / circuit breaker.

### 함정 7 — Keep-alive 의 비대칭
Client ↔ proxy keep-alive. Proxy ↔ backend — proxy_http_version 1.1 + Connection "" 필요.

---

## 15. 학습 자료

- Nginx / HAProxy / Envoy / Caddy / Traefik docs
- "Varnish Cache" docs
- "Building Microservices" (Newman)
- AWS / GCP / Azure API Gateway docs

---

## 16. 관련

- [[cdn-proxy]] — Hub
- [[cdn]] — CDN 의 부모 개념
- [[forward-proxy]] — 반대
- [[../load-balancing/l7-load-balancing]]
- [[../tls-ssl/tls-ssl]] — TLS termination
