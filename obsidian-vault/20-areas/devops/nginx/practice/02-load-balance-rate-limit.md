---
title: "실습 02 — LB + rate-limit + cache (docker-compose)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:34:00+09:00
tags: [devops, nginx, practice, load-balancing, rate-limit]
---

# 실습 02 — LB + rate-limit + cache (docker-compose)

**[[practice|↑ practice]]**

---

## 목표

- nginx 가 3 Spring 인스턴스로 LB
- rate-limit (per IP)
- proxy_cache (1 min)
- log 에서 LB / cache hit 확인

---

## 1. 디렉터리

```
lab-02/
├── docker-compose.yml
├── nginx/
│   ├── nginx.conf
│   └── conf.d/
│       └── app.conf
└── app/
    └── Dockerfile + app.jar
```

---

## 2. docker-compose.yml

```yaml
version: "3.9"
services:
  app1:
    image: demo-web:latest
    environment: {SERVER_PORT: "8080", INSTANCE_ID: "1"}
  app2:
    image: demo-web:latest
    environment: {SERVER_PORT: "8080", INSTANCE_ID: "2"}
  app3:
    image: demo-web:latest
    environment: {SERVER_PORT: "8080", INSTANCE_ID: "3"}

  nginx:
    image: nginx:1.25-alpine
    ports: ["80:80"]
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - nginx-cache:/var/cache/nginx
    depends_on: [app1, app2, app3]

volumes:
  nginx-cache:
```

→ Spring 앱은 `/hello` 응답에 `INSTANCE_ID` 포함하도록 수정.

---

## 3. nginx.conf

```nginx
user nginx;
worker_processes auto;
events { worker_connections 4096; }

http {
    sendfile on;
    keepalive_timeout 65s;

    # rate limit zone
    limit_req_zone $binary_remote_addr zone=api:10m rate=5r/s;

    # cache zone
    proxy_cache_path /var/cache/nginx/cache
        levels=1:2
        keys_zone=my_cache:10m
        max_size=1g
        inactive=10m
        use_temp_path=off;

    log_format main '$remote_addr "$request" $status '
                    'cache=$upstream_cache_status '
                    'upstream=$upstream_addr '
                    'rt=$request_time';
    access_log /var/log/nginx/access.log main;

    include /etc/nginx/conf.d/*.conf;
}
```

---

## 4. conf.d/app.conf

```nginx
upstream backend {
    least_conn;
    server app1:8080 max_fails=3 fail_timeout=15s;
    server app2:8080 max_fails=3 fail_timeout=15s;
    server app3:8080 max_fails=3 fail_timeout=15s;
    keepalive 32;
}

server {
    listen 80;
    server_name _;

    location / {
        limit_req zone=api burst=10 nodelay;
        limit_req_status 429;

        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_cache my_cache;
        proxy_cache_valid 200 1m;
        proxy_cache_key "$scheme$host$request_uri";
        proxy_cache_lock on;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        add_header X-Cache-Status $upstream_cache_status;
    }

    location = /healthz {
        access_log off;
        return 200 "ok";
    }

    location /admin/ {
        allow 127.0.0.1;
        allow 10.0.0.0/8;
        deny all;
        proxy_pass http://backend;
    }
}
```

---

## 5. 실행

```bash
docker compose up -d
docker compose logs -f nginx
```

---

## 6. 검증

### LB 확인

```bash
for i in {1..10}; do
    curl -s http://localhost/hello
    echo
done
```

→ INSTANCE_ID 1/2/3 모두 응답 (least_conn).

### rate-limit 확인

```bash
for i in {1..30}; do
    curl -s -o /dev/null -w "%{http_code}\n" http://localhost/hello
done
```

→ 약 15개 200, 나머지 429.

### cache 확인

```bash
curl -I http://localhost/hello
# X-Cache-Status: MISS
curl -I http://localhost/hello
# X-Cache-Status: HIT
```

### nginx log

```bash
docker compose logs nginx | grep -E "cache=|status=429"
```

```
1.2.3.4 "GET /hello HTTP/1.1" 200 cache=MISS upstream=172.18.0.3:8080 rt=0.012
1.2.3.4 "GET /hello HTTP/1.1" 200 cache=HIT upstream=- rt=0.000
```

---

## 7. failover 시뮬레이션

```bash
docker compose stop app1
for i in {1..10}; do curl -s http://localhost/hello; echo; done
```

→ INSTANCE_ID 2/3 만 응답 (app1 max_fails 도달 후 fail_timeout 동안 skip).

```bash
docker compose start app1
```

---

## 8. 부하

```bash
docker run --rm --network host williamyeh/hey \
    hey -z 30s -c 50 http://localhost/hello
```

→ LB / cache hit rate / rate limit 통계 확인.

---

## 9. 검증 체크리스트

- [ ] 3 instance 모두 응답 (least_conn 분산)
- [ ] rate limit 30 req → 약 15 success, 나머지 429
- [ ] X-Cache-Status MISS → HIT
- [ ] app1 stop 시 자동 failover
- [ ] /admin/ → 외부에서 403, 내부에서 OK

---

## 10. 정리

```bash
docker compose down -v
```

---

## 11. 관련

- [[practice|↑ practice]]
- [[../load-balancing]]
- [[../rate-limiting]]
- [[../caching]]
- [[01-spring-reverse-proxy-tls]]
