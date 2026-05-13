---
title: "L7 Load Balancing — HTTP / Application"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T06:10:00+09:00
tags:
  - network
  - load-balancing
  - l7
  - http
---

# L7 Load Balancing — HTTP / Application

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | ALB / Nginx / Envoy / 라우팅 |

**[[load-balancing|↑ Load Balancing]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

**HTTP 헤더 / URL / Cookie** 를 보고 라우팅. L4 보다 느리지만 풍부한 결정.

---

## 2. 동작

```
Client → LB: HTTP request
LB: 헤더 / URL parsing
    Host: example.com
    Path: /api/...
    Cookie: ...
LB: rule 매칭 → backend 선택
LB → Backend: HTTP forward
Backend → LB → Client: 응답
```

---

## 3. 라우팅 — 기준

| 기준 | 예 |
| --- | --- |
| **Host** | api.example.com → API backend |
| **Path** | /api/* → API, /static/* → CDN |
| **Method** | POST → write replica |
| **Header** | X-Tenant: ... |
| **Cookie** | sticky session |
| **Query param** | ?version=v2 → 새 backend |
| **Client IP** | geographic routing |
| **User-Agent** | mobile vs desktop |
| **SNI** | TLS 의 SNI |

---

## 4. AWS ALB — Application Load Balancer

### 특징
- L7 (HTTP / HTTPS / gRPC / WebSocket)
- Path / Host 기반 라우팅
- HTTP/2 / HTTP/3 (2024)
- WAF / Shield 통합

### 라우팅
```
Listener (443)
   ↓
Rule 1: Host=api.example.com → Target Group A
Rule 2: Path=/admin → Target Group B
Default: → Target Group C
```

### 한계
- IP 고정 X (DNS 만)
- 정적 IP — NLB 권장

---

## 5. Nginx

### 정의
- 2004, Igor Sysoev
- 비동기 / event-driven
- HTTP / HTTPS / reverse proxy / cache

### 설정
```nginx
upstream backend {
    server 10.0.0.1:80 weight=2;
    server 10.0.0.2:80 weight=1;
    server 10.0.0.3:80 backup;
    least_conn;
    keepalive 32;
}

server {
    listen 443 ssl http2;
    server_name example.com;
    ssl_certificate cert.pem;
    ssl_certificate_key key.pem;

    location /api/ {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
    }

    location /static/ {
        root /var/www;
        expires 1y;
    }
}
```

### 알고리즘
- round_robin (기본)
- least_conn
- ip_hash — sticky by IP
- hash $key — consistent hash

---

## 6. HAProxy

### 정의
- 2001, Willy Tarreau
- L4 + L7 둘 다
- 매우 빠름 / 안정

### 설정
```
frontend http_front
    bind *:80
    bind *:443 ssl crt /etc/cert.pem
    redirect scheme https code 301 if !{ ssl_fc }
    
    acl is_api path_beg /api
    use_backend api_backend if is_api
    default_backend web_backend

backend api_backend
    balance leastconn
    option httpchk GET /health
    server api1 10.0.0.1:8080 check
    server api2 10.0.0.2:8080 check

backend web_backend
    balance roundrobin
    server web1 10.0.0.3:80 check
    server web2 10.0.0.4:80 check
```

---

## 7. Envoy

### 정의
- Lyft (2016) — 모던 / 클라우드 native
- C++ — 빠르고 동적 설정
- Service Mesh (Istio / Consul) 의 데이터 plane

### 특징
- xDS API — dynamic configuration
- HTTP/1.1, HTTP/2, HTTP/3, gRPC
- 풍부한 observability
- mTLS / circuit breaker / retry / timeout

### 설정 (예)
```yaml
listeners:
- name: listener_0
  address: { socket_address: { address: 0.0.0.0, port_value: 443 } }
  filter_chains:
  - filters:
    - name: envoy.filters.network.http_connection_manager
      typed_config:
        route_config:
          virtual_hosts:
          - name: backend
            domains: ["*"]
            routes:
            - match: { prefix: "/api" }
              route: { cluster: api_cluster }
            - match: { prefix: "/" }
              route: { cluster: web_cluster }

clusters:
- name: api_cluster
  type: STRICT_DNS
  lb_policy: LEAST_REQUEST
  load_assignment:
    cluster_name: api_cluster
    endpoints:
    - lb_endpoints:
      - endpoint: { address: { socket_address: { address: api1, port_value: 8080 } } }
```

---

## 8. Traefik — Kubernetes 친화

### 정의
- 2015, Containous
- Auto-discovery (Docker / K8s / Consul)
- 모던 dashboard
- Let's Encrypt 자동

### 사용
- Kubernetes Ingress / Gateway API
- Docker Swarm

---

## 9. 라우팅 — 모던 (Service Mesh)

### Service Mesh
- Sidecar pattern — 각 service 의 옆에 Envoy
- Istio / Linkerd / Consul Connect

### 효과
- 보안 (mTLS)
- 관측 (metrics / trace)
- 라우팅 (canary, blue-green)
- Retry / circuit breaker

자세히 → [[../topics/service-mesh]] (TBD)

---

## 10. Canary / Blue-Green

### Canary
```
99% → old version
1% → new version (관찰)
점진 — 10% → 50% → 100%
```

### Blue-Green
```
Blue: 현재 v1
Green: 새 v2
LB switch → Green (즉시)
Rollback — Blue 로 즉시
```

### A/B Testing
- 헤더 / cookie / query 기반
- 트래픽 비율 분산

---

## 11. WebSocket / gRPC

### WebSocket
- L7 LB 가 Upgrade 헤더 처리
- Long-lived connection — sticky / drain 주의

### gRPC
- HTTP/2 multi-stream
- LB 가 stream 단위 분산 (모던) — ALB / Envoy
- 옛 L7 LB — connection 단위만

---

## 12. WAF — Web Application Firewall

### 정의
- L7 의 보안 layer
- SQL injection / XSS / OWASP Top 10 검출

### 제품
- AWS WAF
- Cloudflare WAF
- ModSecurity (open source)
- Imperva

### LB 와 통합
- ALB → AWS WAF
- Cloudflare — 자체

---

## 13. 함정

### 함정 1 — X-Forwarded-For chain
LB 가 여러 — 헤더 chain. 신뢰 가능한 LB 까지만 trust.

### 함정 2 — Sticky session 의 scale-down
한 backend 죽으면 session 분산. Stateless + Redis 세션 권장.

### 함정 3 — HTTP/2 의 multiplexing
한 connection 의 여러 stream — connection 단위 분산은 비효율.
Stream-aware LB (Envoy / ALB) 사용.

### 함정 4 — TLS termination 의 backend 신뢰
LB ↔ backend — 평문이면 내부 망 보안 가정.
mTLS 권장.

### 함정 5 — Connection re-use 의 균등 분산
keepalive — 같은 backend 재사용 → 균등 분산 깨짐.
주기적 connection refresh.

### 함정 6 — Path / Host 의 모호함
"/api" 와 "/api/" 차이. 정규식 / 정확 매칭 확인.

---

## 14. 학습 자료

- AWS ALB / Nginx / HAProxy / Envoy / Traefik docs
- "Envoy Internals" (Lyft 블로그)
- "Service Mesh Comparison" 등

---

## 15. 관련

- [[load-balancing]] — Hub
- [[l4-load-balancing]] — 비교
- [[../cdn-proxy/reverse-proxy]] — L7 의 다른 이름
- [[../http/versions/version-comparison]] — HTTP/2 / HTTP/3
