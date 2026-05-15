---
title: "Load balancing — upstream + 알고리즘"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:15:00+09:00
tags: [devops, nginx, load-balancing]
---

# Load balancing — upstream + 알고리즘

**[[nginx|↑ nginx]]**

---

## 1. 기본

```nginx
upstream backend {
    server app1.internal:8080;
    server app2.internal:8080;
    server app3.internal:8080;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

→ default = **round-robin**.

---

## 2. 알고리즘

```nginx
# round-robin (default)
upstream backend { server a; server b; }

# weighted round-robin
upstream backend {
    server a weight=3;        # 75%
    server b weight=1;        # 25%
}

# least connections
upstream backend {
    least_conn;
    server a;
    server b;
}

# IP hash (session affinity)
upstream backend {
    ip_hash;                  # 같은 client IP → 같은 server
    server a;
    server b;
}

# hash (key 기반 — consistent hash)
upstream backend {
    hash $request_uri consistent;
    server a;
    server b;
}

# random
upstream backend {
    random two least_conn;    # 2개 무작위 + 둘 중 적은 연결
    server a;
    server b;
}
```

---

## 3. 언제 어떤 알고리즘

| 상황 | 알고리즘 |
| --- | --- |
| 일반 stateless 서비스 | round-robin |
| backend 성능 다름 | weight |
| backend response time 다름 | least_conn |
| sticky session 필요 | ip_hash / cookie hash |
| cache hit 최대화 | hash $request_uri consistent |
| 최신 (1.15+) | random two least_conn |

→ **JWT / stateless** 면 ip_hash 불필요. **session 사용** 시만.

---

## 4. server 옵션

```nginx
upstream backend {
    server a:8080 weight=3 max_fails=3 fail_timeout=30s;
    server b:8080 weight=1;
    server c:8080 backup;             # primary 모두 fail 시
    server d:8080 down;               # 임시 비활성
    server e:8080 max_conns=100;      # max concurrent
    server unix:/tmp/backend.sock;    # Unix socket
}
```

| 옵션 | 의미 |
| --- | --- |
| `weight=N` | 가중치 (default 1) |
| `max_fails=N` | N번 실패 시 down |
| `fail_timeout=T` | down 으로 표시 후 T 동안 skip |
| `backup` | primary 모두 fail 시만 사용 |
| `down` | 영구 비활성 |
| `max_conns=N` | 동시 연결 제한 |

---

## 5. health check (active — nginx Plus / OpenResty)

```nginx
# OSS nginx: passive only (요청 실패 시 mark down)

# Plus: active health check
upstream backend {
    zone backend 64k;
    server a:8080;
    server b:8080;
}

server {
    location / {
        proxy_pass http://backend;
        health_check interval=5s uri=/health match=ok;
    }
}

match ok {
    status 200;
    body ~ "OK";
}
```

→ OSS 대안: OpenResty + lua, 또는 ALB / Envoy.

---

## 6. session affinity (sticky)

```nginx
# IP hash (간단)
upstream backend {
    ip_hash;
    server a; server b;
}

# 단점: NAT / proxy 뒤 → 한 IP 에 다 몰림

# cookie 기반 (Plus 만)
upstream backend {
    sticky cookie srv_id expires=1h domain=.example.com path=/;
    server a; server b;
}
```

→ **stateless JWT 면 affinity 필요 X**.

---

## 7. consistent hash (cache 친화)

```nginx
upstream cache_backend {
    hash $request_uri consistent;
    server c1; server c2; server c3;
}
```

→ server 추가/제거 시 키 재분배 최소 (전통 hash 는 90% 재분배).

---

## 8. dynamic upstream (k8s 등)

```nginx
# 정적 — config reload 필요
upstream backend {
    server pod1:8080;
    server pod2:8080;
}

# DNS 기반 (k8s service)
upstream backend {
    server backend.default.svc.cluster.local:8080 resolve;
    # ★ resolve 옵션 = nginx Plus
}

# OSS 대안: 변수 사용
resolver 10.96.0.10 valid=10s;
location / {
    set $upstream "backend.default.svc.cluster.local";
    proxy_pass http://$upstream:8080;
}

# 또는 nginx-ingress / Traefik 사용
```

---

## 9. zone (shared memory)

```nginx
upstream backend {
    zone backend 64k;             # ★ 필요 (Plus 의 dynamic / health_check)
    server a; server b;
}
```

→ worker 간 상태 공유. health check / sticky / dynamic 에 필요.

---

## 10. L4 (TCP/UDP) load balance

```nginx
# /etc/nginx/nginx.conf
stream {
    upstream postgres {
        least_conn;
        server pg1:5432 max_fails=3 fail_timeout=30s;
        server pg2:5432;
    }

    server {
        listen 5432;
        proxy_pass postgres;
        proxy_timeout 1h;
    }
}
```

→ HTTP 가 아닌 TCP/UDP. nginx 가 HAProxy 대안 가능.

---

## 11. 함정

1. **ip_hash + 모든 client 가 같은 IP** (NAT 뒤) — 한 server 로 몰림.
2. **fail_timeout 너무 길게** — 복구 후에도 traffic 안 옴.
3. **max_fails=0** — 영원히 안 down.
4. **DNS resolve 한 번만** — k8s pod 변동 시 stale.
5. **session affinity + stateless 앱** — 필요 없는 hotspot.
6. **OSS 의 health check 가 진짜 active** 라 오해 — passive only.

---

## 12. 관련

- [[nginx|↑ nginx]]
- [[reverse-proxy]]
- [[concepts]]
- [[../kubernetes/ingress-controllers|↗ k8s ingress]]
