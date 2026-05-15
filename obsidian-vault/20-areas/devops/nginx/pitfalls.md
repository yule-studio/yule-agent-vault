---
title: "nginx — 함정 모음"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:29:00+09:00
tags: [devops, nginx, pitfalls]
---

# nginx — 함정 모음

**[[nginx|↑ nginx]]**

---

## 1. config

1. **`nginx -t` 안 함** — broken config 로 reload → 운영 영향.
2. **reload 안 함** — 변경 후 적용 안 됨.
3. **sites-enabled/default 남음** — server_name 매칭 X.
4. **default_server 두 번** — error.
5. **server_name 따옴표** — `server_name "x";` ❌, `server_name x;`.
6. **listen IPv6 (`[::]:80`) 빠짐** — IPv6 client 안 됨.
7. **`if` 남용** — "if is evil" docs, 의도와 다른 동작.

---

## 2. location

1. **regex 우선순위 오해** — exact > `^~` > regex > prefix.
2. **trailing slash 차이** — `/api` 와 `/api/` 다름.
3. **proxy_pass trailing slash** — URI rewrite 됨.
4. **`location /` catch-all 안 둠** — 404 처리 X.

---

## 3. reverse proxy

1. **Host header 안 넘김** — backend 가 잘못된 host.
2. **X-Forwarded-For spoofable** — real_ip_header + set_real_ip_from.
3. **keepalive Connection 미설정** — 매 요청 new TCP.
4. **DNS resolve 한 번만** — 변수 또는 resolver 사용.
5. **timeout 60s default** — long-poll / SSE 끊김.
6. **proxy_buffering on + SSE** — chunk 늦게.

---

## 4. load balancing

1. **ip_hash + NAT** — 한 서버로 몰림.
2. **fail_timeout 너무 길게** — 복구 후에도 traffic 안 옴.
3. **max_fails=0** — 영원히 안 down.
4. **DNS stale** — k8s pod 변동 시.
5. **session affinity + stateless** — hotspot.
6. **OSS active health check** 오해 — passive only.

---

## 5. TLS

1. **HSTS preload 잘못 등록** — 1년 잠김.
2. **TLS 1.0/1.1 켜둠** — 취약.
3. **자체서명 cert** — 브라우저 경고.
4. **갱신 hook 없음** — cert 갱신 후 옛 cert.
5. **/.well-known/acme-challenge block** — 갱신 실패.
6. **intermediate 빠짐** — 일부 client cert 검증 실패.
7. **rate limit (LE)** — 같은 도메인 주 5회 한도.

---

## 6. cache

1. **★ cache key 에 user 정보 누락** — 다른 사용자 데이터 누출.
2. **POST cache** — 보통 안 함.
3. **set-cookie 응답 cache** — 모든 user 동일 cookie.
4. **무한 cache** — backend 변경 안 반영.
5. **disk full** — max_size + 모니터링.
6. **cache lock + slow backend** — timeout.

---

## 7. rate-limit

1. **$remote_addr 가 ALB IP** — real_ip 안 함 → 무의미.
2. **너무 엄격** — 정상 사용자 429.
3. **burst 너무 큼** — backend 폭주.
4. **zone memory 부족** — shared memory too small.
5. **POST body 후 limit** — 보호 의미 ↓.
6. **login brute force 와 일반 혼동** — login 별도 엄격 zone.

---

## 8. 보안

1. **HSTS preload 미숙 등록** — 1년 lock.
2. **CSP 너무 엄격** — site 깨짐. Report-Only 먼저.
3. **server_tokens on** — version 노출.
4. **WAF false positive** — 정상 traffic 차단.
5. **`add_header always` 누락** — 4xx 응답에서 빠짐.
6. **`if` 남용** — 의도 X.

---

## 9. performance

1. **worker_processes 1** — 1 core만.
2. **keepalive upstream 없음** — backend connection 폭증.
3. **buffer 부족** — disk spill.
4. **gzip on + HTTP/1.0** — 미지원 client.
5. **HTTP/2 + 작은 file** — 오히려 느림.
6. **access_log 모두 켜둠** — disk IO.

---

## 10. 운영

1. **graceful reload 안 함** — `nginx -s reload` 대신 restart → 끊김.
2. **error log 모니터링 안 함** — 침묵의 fail.
3. **rotate 안 함** — log 무한 누적.
4. **권한 — config 644 / private key 600** — 잘못 시 보안 노출.
5. **stub_status 외부 노출** — server info 누출.

---

## 11. 관련

- [[nginx|↑ nginx]]
