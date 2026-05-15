---
title: "nginx rate-limiting — limit_req / limit_conn"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:22:00+09:00
tags: [devops, nginx, rate-limit]
---

# nginx rate-limiting — limit_req / limit_conn

**[[nginx|↑ nginx]]**

---

## 1. 왜

- DDoS / brute force 차단.
- 외부 API 호출량 제한 (downstream 보호).
- abuse / scraping 방지.

→ application 단 [[../../../60-recipes/spring/rate-limiting/rate-limiting|Bucket4j]] 와 다른 layer. nginx 가 first defense.

---

## 2. limit_req (request rate)

```nginx
# http {} context
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
# zone 이름: api, 메모리 10MB (≈ 160,000 IP), 초당 10 request

server {
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://backend;
    }
}
```

| 옵션 | 의미 |
| --- | --- |
| `rate=10r/s` | 평균 초당 10 (token bucket) |
| `burst=20` | 짧은 시간 폭주 허용 (queue) |
| `nodelay` | burst 내 즉시 처리 (default = delay) |
| `delay=10` (1.15+) | 10 까지 즉시, 그 이상 delay |

---

## 3. limit_req_zone key 옵션

```nginx
# IP
limit_req_zone $binary_remote_addr zone=ip:10m rate=10r/s;

# 사용자 ID (header / cookie)
limit_req_zone $http_x_user_id zone=user:10m rate=100r/m;

# API key
limit_req_zone $http_x_api_key zone=key:10m rate=1000r/h;

# URI (per path)
limit_req_zone $request_uri zone=uri:10m rate=5r/s;

# 복합
limit_req_zone "$binary_remote_addr$request_uri" zone=ip_uri:10m rate=1r/s;
```

→ `$binary_remote_addr` 가 `$remote_addr` 보다 효율 (4-byte vs string).

---

## 4. burst / nodelay 동작

```
rate=10r/s, burst=20, nodelay

t=0:   100 requests come
       → 1-20 즉시 처리 (burst)
       → 21-100 → 503

t=0~1s: 추가 10 request 처리 가능 (rate)
```

```
rate=10r/s, burst=20 (no nodelay)

t=0:   100 requests come
       → 1-20 queue 에 들어감 (각 0.1초 간격으로 처리)
       → 21-100 → 503
```

→ **nodelay 가 일반적**. queueing latency 없이 burst 허용.

---

## 5. limit_conn (concurrent connection)

```nginx
limit_conn_zone $binary_remote_addr zone=conn_ip:10m;

server {
    location / {
        limit_conn conn_ip 10;       # IP 당 동시 10 connection
        proxy_pass http://backend;
    }
}
```

→ slowloris / 다운로드 abuse 방지.

---

## 6. 다단계 (global + per-endpoint)

```nginx
# 전체 limit
limit_req_zone $binary_remote_addr zone=global:10m rate=100r/s;
# /login limit (brute force)
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

server {
    location / {
        limit_req zone=global burst=200 nodelay;
        proxy_pass http://backend;
    }

    location = /login {
        limit_req zone=login burst=3 nodelay;
        proxy_pass http://backend;
    }

    location = /signup {
        limit_req zone=login burst=3 nodelay;
        proxy_pass http://backend;
    }
}
```

---

## 7. error 응답 / log

```nginx
limit_req_status 429;            # default 503 → 429 (Too Many Requests)
limit_conn_status 429;

limit_req_log_level warn;        # rate limit 로그 level

# header
add_header X-RateLimit-Limit 10;
add_header X-RateLimit-Remaining $limit_req_status;   # 표준 X
```

→ standard rate limit header 는 application 에서.

---

## 8. whitelist (allow / geo)

```nginx
# IP whitelist
geo $limit {
    default 1;                    # rate limit 적용
    10.0.0.0/8 0;                 # 내부망 → 적용 안 함
    127.0.0.1 0;
    192.168.0.0/16 0;
}

map $limit $limit_key {
    0 "";                         # 빈 key → limit_req 무시
    1 $binary_remote_addr;
}

limit_req_zone $limit_key zone=api:10m rate=10r/s;
```

---

## 9. CDN / proxy 뒤 (real_ip)

```nginx
# Cloudflare / ALB 뒤 → $remote_addr 는 proxy IP
# → real client IP 가져오려면 X-Forwarded-For 신뢰

set_real_ip_from 10.0.0.0/8;     # 신뢰 IP 범위
real_ip_header X-Forwarded-For;
real_ip_recursive on;

# 이제 $remote_addr 가 진짜 client IP
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
```

→ 안 하면 모든 traffic 이 ALB IP 1개로 보임 → rate limit 무의미.

---

## 10. Cloudflare / ALB 단에서도 함

nginx 만으로는 부족:
- DDoS (volumetric) — 이미 server 까지 와서 처리.
- 진짜 큰 공격 → upstream (Cloudflare, AWS Shield).
- nginx rate limit = abuse / brute force / 일반 보호.

---

## 11. 함정

1. **$remote_addr 가 ALB IP** — proxy 뒤 real_ip 안 함.
2. **너무 엄격** — 정상 사용자도 429.
3. **burst 너무 큼** — backend 폭주.
4. **zone memory 부족** — `nginx[...]: 'api' shared memory zone is too small`.
5. **POST body 받은 후 limit** — body 까지 받은 후 reject → 의미 ↓ (요청 보호 X).
6. **rate-limit + brute force protection 혼동** — login 은 더 엄격하게 별도 zone.

---

## 12. 관련

- [[nginx|↑ nginx]]
- [[security]]
- [[../../60-recipes/spring/rate-limiting/rate-limiting|↗ app rate-limit]]
