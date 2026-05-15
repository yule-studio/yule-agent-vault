---
title: "nginx 핵심 개념 — master/worker"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:07:00+09:00
tags: [devops, nginx, concepts]
---

# nginx 핵심 개념 — master/worker

**[[nginx|↑ nginx]]**

---

## 1. 왜 빠른가

```
Apache (prefork): 1 connection = 1 process
                  → 10,000 connection = 10,000 process → 메모리 폭주

nginx (event-driven): 1 process = N connections (epoll/kqueue)
                      → 10,000 connection = 1 process (CPU 만 충분하면)
```

→ **non-blocking IO + event loop**.

---

## 2. master + worker

```
[master process]    root, 단일
  │ listen socket bind
  │ config 검증 / reload
  │
  ├─ [worker 1]    nobody (또는 nginx user)
  │   event loop, N connection 처리
  ├─ [worker 2]
  └─ [worker N]   (보통 CPU core 수)
```

```nginx
# nginx.conf
worker_processes auto;       # = core 수
worker_connections 1024;     # worker 당 max connection
```

→ max connection = `worker_processes × worker_connections`.

---

## 3. event loop

```
1. accept() new connection → epoll 에 등록
2. epoll_wait() 로 IO ready event 대기
3. ready 되면 처리 (read / parse / send)
4. 처리 끝나면 next iteration
```

→ 한 worker 가 수천 connection 동시 처리.

---

## 4. signal 동작

| Signal | 무엇 |
| --- | --- |
| **SIGHUP** | config reload (graceful) — `nginx -s reload` |
| **SIGUSR1** | log rotation — `nginx -s reopen` |
| **SIGUSR2** | binary upgrade (hot upgrade) |
| **SIGQUIT** | graceful shutdown — `nginx -s quit` |
| **SIGTERM** | fast shutdown — `nginx -s stop` |

→ `nginx -s reload` 는 master 가 새 worker 띄우고 기존 worker 는 처리 끝나면 종료. 무중단.

---

## 5. context

```nginx
# 최상위 (main)
worker_processes auto;
error_log /var/log/nginx/error.log;

# events context
events {
    worker_connections 1024;
    use epoll;
}

# http context (HTTP 관련)
http {
    include mime.types;
    sendfile on;

    # server context (가상 호스트)
    server {
        listen 80;
        server_name example.com;

        # location context (URI 패턴)
        location /api/ {
            proxy_pass http://backend;
        }
    }
}

# stream context (TCP/UDP — DB proxy 등)
stream {
    server {
        listen 5432;
        proxy_pass postgres-cluster;
    }
}
```

→ http / stream / events / main 4가지 top-level.

---

## 6. directive 종류

```
simple:     name value;            # 한 줄, ; 로 끝
            worker_processes auto;
            listen 80;

block:      name { ... }           # 중첩
            server { listen 80; }
            location / { ... }
```

context 별로 가능한 directive 다름 (`http {}` 안에서만, `location {}` 안에서만 등).

---

## 7. include

```nginx
http {
    include /etc/nginx/conf.d/*.conf;        # 분리된 server block
    include /etc/nginx/snippets/*.conf;       # 재사용 partial
}
```

→ `/etc/nginx/sites-available/` + `sites-enabled/` 패턴 (Debian) — symlink 로 enable.

---

## 8. 변수

```nginx
# 내장 변수
$remote_addr        # client IP
$request_uri        # 전체 URI
$host               # Host header
$args               # query string
$arg_NAME           # 특정 query param ($arg_id)
$http_NAME          # header ($http_user_agent)
$cookie_NAME        # cookie
$scheme             # http / https
$request_method     # GET / POST
$server_name        # server_name 디렉티브
$status             # response status (log 용)
$body_bytes_sent
```

```nginx
# 사용 예
location / {
    if ($http_user_agent ~* "(bot|spider)") {
        return 403;
    }
    proxy_set_header X-Real-IP $remote_addr;
    return 200 "Hello $host, path=$request_uri";
}
```

---

## 9. config 검증 / reload

```bash
sudo nginx -t                      # syntax check (★ 항상 먼저)
sudo nginx -T                      # 전체 dump
sudo nginx -s reload               # graceful reload (SIGHUP)
sudo nginx -s stop                 # 즉시 stop
sudo systemctl restart nginx       # 또는
```

---

## 10. 함정

1. **reload 안 함** — 변경 후 `nginx -s reload` 안 하면 적용 X.
2. **`nginx -t` 안 함** — broken config 로 reload 시 운영 영향.
3. **worker_connections 너무 작음** — connection refused.
4. **`if` 남용** — 의도와 다르게 동작 ("if is evil" docs).
5. **regex location** 순서 — exact > prefix > regex.
6. **`/etc/nginx/sites-enabled/default`** 남아있음 — default server 가 우선.

---

## 11. 관련

- [[nginx|↑ nginx]]
- [[config-structure]]
- [[reverse-proxy]]
