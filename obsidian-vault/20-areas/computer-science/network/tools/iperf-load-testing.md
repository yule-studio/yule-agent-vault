---
title: "iperf / wrk / k6 — 대역 / 부하 측정"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:20:00+09:00
tags:
  - network
  - tools
  - benchmarking
  - load-testing
---

# iperf / wrk / k6 — 대역 / 부하 측정

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | iperf / wrk / hey / k6 / Locust |

**[[tools|↑ 도구 hub]]** · **[[../network|↑↑ network hub]]**

---

## 1. iperf3 — 대역 측정

### 정의
- 네트워크의 max 대역 측정 (TCP / UDP)
- 클라이언트 / 서버 모드

### 서버
```bash
iperf3 -s
# Listening on 5201
```

### 클라이언트
```bash
iperf3 -c server.example.com

# 결과
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-10.00 sec   1.10 GBytes   946 Mbits/sec
```

### 옵션
```bash
# 시간
iperf3 -c server -t 30          # 30 초

# 병렬
iperf3 -c server -P 4           # 4 stream

# UDP
iperf3 -c server -u -b 100M     # 100 Mbps

# 역방향 (server → client)
iperf3 -c server -R

# JSON 출력
iperf3 -c server -J
```

### 사용
- 네트워크 link 검증
- VPN throughput
- 클라우드 region 간 대역

---

## 2. wrk — HTTP 부하

### 정의
- Will Glozer (2012)
- C + Lua, 매우 빠름
- 단일 server 부담

### 기본
```bash
wrk -t 4 -c 100 -d 30s https://example.com

# -t 4: 4 thread
# -c 100: 100 connection
# -d 30s: 30 초

# 결과
Running 30s test @ https://example.com
  4 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    10.5ms    2.3ms    50.0ms   80.0%
    Req/Sec    2.4k      150     2.8k     85.0%
  288000 requests in 30.01s, 245.2MB read
Requests/sec: 9596.27
Transfer/sec: 8.17MB
```

### Lua 스크립트
```lua
-- post.lua
wrk.method = "POST"
wrk.headers["Content-Type"] = "application/json"
wrk.body = '{"name": "alice"}'
```

```bash
wrk -s post.lua -t 4 -c 100 -d 30s https://api/users
```

---

## 3. hey — Go 작성

### 정의
- Apache ab 의 모던 대체
- Go 작성

```bash
hey -n 1000 -c 50 https://example.com
# -n: total request
# -c: 동시

# 결과
Summary:
  Total:        2.5 sec
  Slowest:      0.3 sec
  Fastest:      0.05 sec
  Average:      0.12 sec
  Requests/sec: 400

Latency distribution:
  50% in 0.1s
  90% in 0.2s
  99% in 0.28s

Status code distribution:
  [200] 1000 responses
```

---

## 4. ab — Apache Bench (옛)

```bash
ab -n 1000 -c 50 https://example.com/

# 단순 / 기본
# Limitations: 옛 HTTP/1.0, no keep-alive 기본
```

---

## 5. k6 — 모던 스크립트 부하

### 정의
- Grafana k6 (Load Impact)
- JavaScript 스크립트
- Cloud / OSS

### 스크립트
```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
    stages: [
        { duration: '30s', target: 100 },    // ramp up
        { duration: '1m',  target: 100 },    // sustain
        { duration: '30s', target: 0 },      // ramp down
    ],
    thresholds: {
        http_req_duration: ['p(95)<500'],
        http_req_failed:   ['rate<0.01'],
    },
};

export default function () {
    const res = http.get('https://example.com');
    check(res, {
        'status 200': (r) => r.status === 200,
        'fast': (r) => r.timings.duration < 500,
    });
    sleep(1);
}
```

### 실행
```bash
k6 run script.js
k6 run --vus 100 --duration 30s script.js
k6 cloud script.js              # Grafana Cloud
```

### 장점
- 시나리오 (단계별 ramp)
- Threshold (자동 pass/fail)
- 메트릭 / dashboard

---

## 6. Locust — Python 부하

### 정의
- 분산 / Python 친화
- Web UI

```python
from locust import HttpUser, task, between

class WebUser(HttpUser):
    wait_time = between(1, 3)
    
    @task
    def index(self):
        self.client.get('/')
    
    @task(3)        # 3x weight
    def view_item(self):
        self.client.get('/items/1')
```

```bash
locust -f locust.py --host=https://example.com
# Web UI: localhost:8089
```

---

## 7. ghz — gRPC 부하

```bash
ghz --insecure \
    --proto user.proto \
    --call user.UserService/GetUser \
    -d '{"id": 1}' \
    -n 10000 -c 50 \
    grpc-server:50051
```

---

## 8. JMeter — GUI / 엔터프라이즈

### 정의
- Apache JMeter
- Java GUI / 분산

### 사용
- 큰 / 복잡한 시나리오
- 다양한 프로토콜 (HTTP / JMS / DB / FTP / SOAP)
- CI 통합

### 단점
- GUI 무거움
- 옛 — k6 / Gatling 권장

---

## 9. Gatling — Scala DSL

```scala
class BasicSimulation extends Simulation {
    val httpProtocol = http.baseUrl("https://example.com")
    
    val scn = scenario("Test")
        .exec(http("home").get("/"))
        .pause(1)
    
    setUp(scn.inject(rampUsers(100) during 10.seconds).protocols(httpProtocol))
}
```

---

## 10. vegeta — Go / 분산

```bash
echo "GET https://example.com" | vegeta attack -duration=30s -rate=1000 | vegeta report
```

### 장점
- 표준 입력 — 다양한 endpoint
- Distributed 가능

---

## 11. iperf2 vs iperf3

| | iperf2 | iperf3 |
| --- | --- | --- |
| 병렬 stream | 좋음 | 좋음 |
| Bidirectional | 좋음 | 한 번에 한 방향 (옛 한계) |
| JSON | X | O |
| Multi-thread | X | X (single-thread) |
| 사용 | 옛 / 복잡한 시나리오 | 모던 / 일반 |

---

## 12. 흔한 측정 지표

### Latency / Response time
- Avg / median / p95 / p99 / p99.9
- p99 이 중요 (꼬리 latency)

### Throughput
- req/s (HTTP)
- Mbps (raw bandwidth)

### Error rate
- % of 5xx, timeout
- Threshold (예: <0.1%)

### Saturation
- CPU / memory / connection / file descriptor
- 한계 도달 점

---

## 13. 부하 테스트 — 시나리오

### Smoke test
- 1-2 사용자, 짧음
- 기본 동작 확인

### Load test
- 일반 사용자 수 (예: 100)
- 평소 부하

### Stress test
- 한계 찾기 (200 → 500 → 1000)
- Breaking point

### Spike test
- 갑작스러운 polling (예: 100 → 10000)
- 마케팅 / 행사

### Soak test
- 오래 (24h+)
- Memory leak / GC pause

### Capacity test
- 목표 부하 — 견디는지

---

## 14. 함정

### 함정 1 — Tool 자체가 병목
wrk / hey — 단일 server. 큰 부하 — 분산 (k6 / vegeta).

### 함정 2 — Localhost 측정
지역 측정 — 네트워크 latency 없음. 실제 환경에서.

### 함정 3 — Warm-up 없음
첫 요청 — connection / TLS / JIT 미초기화. Ramp up.

### 함정 4 — Keep-alive
HTTP — keep-alive 활성. 옛 도구 (ab) 는 매번 connect.

### 함정 5 — Throughput vs Latency
trade-off. 둘 다 측정.

### 함정 6 — DNS / TLS 의 캐시
같은 host 반복 — cache hit. 다양한 url 권장.

### 함정 7 — Server 와 LB 의 분리 측정
LB / proxy / origin — 어디가 병목? 각 layer 측정.

---

## 15. 학습 자료

- "Systems Performance" (Gregg) — benchmarking
- k6 / Gatling / Locust docs
- "Brendan Gregg's Performance Tools"
- web.dev — Performance budgets

---

## 16. 관련

- [[tools]] — Hub
- [[../load-balancing/health-check]] — Outlier detect
- [[../topics/c10k-c10m]] (TBD) — Throughput 한계
