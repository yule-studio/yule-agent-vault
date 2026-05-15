---
title: "nginx performance tuning"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:27:00+09:00
tags: [devops, nginx, performance]
---

# nginx performance tuning

**[[nginx|↑ nginx]]**

---

## 1. worker

```nginx
# nginx.conf
worker_processes auto;                # = CPU core 수 (자동)
worker_rlimit_nofile 100000;          # worker 당 file descriptor

events {
    worker_connections 10240;         # worker 당 max connection
    use epoll;                         # linux 기본
    multi_accept on;                  # 한 번에 여러 connection accept
}
```

→ max concurrent = `worker_processes × worker_connections`.  
→ worker 가 80% CPU 미만이면 worker 수 그대로 (network/disk bound).

---

## 2. system level (sysctl)

```bash
# /etc/sysctl.conf
net.core.somaxconn = 65535            # listen() backlog
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_tw_reuse = 1
net.core.netdev_max_backlog = 65535
fs.file-max = 2097152

# 적용
sudo sysctl -p
```

→ container 환경 (k8s) 에서는 `initContainer` 로 적용.

---

## 3. keepalive

### client ↔ nginx

```nginx
keepalive_timeout 65s;
keepalive_requests 1000;              # connection 당 max requests
```

### nginx ↔ backend (★ 큰 차이)

```nginx
upstream backend {
    server app1:8080;
    server app2:8080;
    keepalive 32;                     # ★ pool 크기
    keepalive_timeout 60s;
    keepalive_requests 1000;
}

location / {
    proxy_pass http://backend;
    proxy_http_version 1.1;           # ★ HTTP/1.1 (keepalive)
    proxy_set_header Connection "";   # ★ 빈 Connection header
}
```

→ 안 하면 매 요청마다 new TCP → 큰 overhead.

---

## 4. buffer

```nginx
# client request
client_body_buffer_size 16k;
client_max_body_size 10m;
client_header_buffer_size 4k;
large_client_header_buffers 4 16k;

# proxy response
proxy_buffer_size 8k;
proxy_buffers 16 8k;
proxy_busy_buffers_size 16k;

# 임시 파일 (disk)
client_body_temp_path /var/cache/nginx/client_temp;
proxy_temp_path /var/cache/nginx/proxy_temp;
```

→ buffer 부족 → disk 로 spill → 느림.

---

## 5. sendfile / tcp_nopush / tcp_nodelay

```nginx
sendfile on;                          # zero-copy file send (★)
tcp_nopush on;                        # 큰 packet 만 send (sendfile 과 같이)
tcp_nodelay on;                       # 작은 packet 즉시 (Nagle 비활성)
```

→ static file serving 시 sendfile 매우 효과적.

---

## 6. gzip

```nginx
gzip on;
gzip_vary on;
gzip_min_length 1024;
gzip_comp_level 5;                    # 1-9, 5 가 balance
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
gzip_proxied any;

# Brotli (Cloudflare 모듈)
brotli on;
brotli_comp_level 5;
brotli_types text/plain text/css application/json;
```

→ Brotli 가 gzip 보다 약 20% 더 압축. 단 사전 compile 또는 module.

---

## 7. open_file_cache (static)

```nginx
open_file_cache max=10000 inactive=20s;
open_file_cache_valid 30s;
open_file_cache_min_uses 2;
open_file_cache_errors on;
```

→ static file 의 metadata cache (inode, size, mtime). 디스크 stat 절약.

---

## 8. HTTP/2 / HTTP/3

```nginx
# HTTP/2
listen 443 ssl http2;

# HTTP/3 (QUIC, 1.25+)
listen 443 quic reuseport;
listen 443 ssl http2;
add_header Alt-Svc 'h3=":443"; ma=86400';
```

→ HTTP/2 = multiplexing. HTTP/3 = QUIC (UDP, 0-RTT).

---

## 9. logging 최적화

```nginx
# log format buffering
access_log /var/log/nginx/access.log main buffer=64k flush=5s;

# 특정 location 은 access_log off (static)
location ~* \.(js|css|jpg|png|gif|ico|woff2)$ {
    access_log off;
}

# health check log 제외
location = /healthz {
    access_log off;
    return 200 "ok";
}
```

---

## 10. monitoring (stub_status)

```nginx
# OSS — 기본 metric
location /nginx_status {
    stub_status on;
    allow 127.0.0.1;
    deny all;
}
```

```
$ curl localhost/nginx_status
Active connections: 291
server accepts handled requests
 16630948 16630948 31070465
Reading: 6 Writing: 179 Waiting: 106
```

→ Prometheus exporter: `nginx-prometheus-exporter` 사용.

---

## 11. cache (proxy / fastcgi)

→ [[caching]] 참조.

---

## 12. 측정 (load test)

```bash
# wrk (간단)
wrk -t12 -c400 -d30s https://example.com/

# k6 (현대)
k6 run --vus 100 --duration 30s script.js

# bombardier (Go, 빠름)
bombardier -c 100 -d 30s https://example.com/

# Apache Bench (legacy)
ab -n 10000 -c 100 https://example.com/
```

→ 측정 → tuning → 다시 측정 (★ 추측 X).

---

## 13. 흔한 병목 찾기

```
1. CPU bound — worker_processes / cipher (TLS) 무거움
2. Network bound — bandwidth (cloud egress 비싸기도)
3. file descriptor 부족 — worker_rlimit_nofile / ulimit
4. backend slow — proxy_read_timeout / 백엔드 자체 문제
5. disk bound — sendfile / proxy_temp_path SSD
```

---

## 14. 함정

1. **worker_processes 1** — 1 core 만 사용.
2. **keepalive upstream 안 함** — backend connection 폭증.
3. **버퍼 너무 작음** — disk spill 빈번 → 느림.
4. **gzip on 인데 client 에 gzip 미지원** (HTTP/1.0).
5. **HTTP/2 + 작은 file** — 헤더 압축 overhead → HTTP/1.1 보다 느릴 수도.
6. **access_log 모두 켜둠 — disk IO 폭주** — 필요한 경로만.

---

## 15. 관련

- [[nginx|↑ nginx]]
- [[caching]]
- [[reverse-proxy]]
- [[../linux/performance-troubleshooting|↗ linux performance]]
