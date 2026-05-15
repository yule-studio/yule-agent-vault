---
title: "Load testing — k6 / JMeter / wrk / Locust"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:25:00+09:00
tags: [devops, performance, load-testing]
---

# Load testing — k6 / JMeter / wrk / Locust

**[[performance|↑ performance]]**

---

## 1. 종류

| | 무엇 |
| --- | --- |
| **smoke** | 1 user, 정상 동작 확인 |
| **load** | 일반 traffic 시뮬 |
| **stress** | peak / 한계 찾기 |
| **soak (endurance)** | 장시간 (24h) — memory leak |
| **spike** | 갑작스 트래픽 |
| **breakpoint** | 점진 ↑, 어디서 깨지나 |

---

## 2. 도구 비교

| | 강점 | 약점 |
| --- | --- | --- |
| **k6** (★) | JS, modern, 빠름 | Go base, GUI X |
| **JMeter** | GUI, 풍부, 전통 | 무거움, 느림 |
| **wrk** | 매우 빠름, 단순 | scripting 제약 |
| **wrk2** | wrk + 일정 rate | scripting Lua |
| **Locust** | Python, distributed | slow per worker |
| **Gatling** | Scala, 강력 | 학습 곡선 |
| **Vegeta** | Go, CLI, 빠름 | report 약함 |
| **Apache Bench** (ab) | 매우 단순 | basic only |
| **hey** (Go) | 빠르고 단순 | basic |
| **Artillery** | Node.js | 작은 community |

→ **현재 표준 = k6**.

---

## 3. wrk (★ 가장 빠름)

```bash
# 기본
wrk -t12 -c400 -d30s https://api.example.com/

#  -t  threads
#  -c  connections
#  -d  duration

# Lua script
cat > post.lua <<EOF
wrk.method = "POST"
wrk.body = '{"name":"test"}'
wrk.headers["Content-Type"] = "application/json"
EOF
wrk -t8 -c100 -d30s -s post.lua https://api.example.com/users
```

→ 1 thread 가 수천 connection. CPU 가 잠재력.

---

## 4. k6 (★ modern)

```javascript
// script.js
import http from 'k6/http'
import { check, sleep } from 'k6'
import { Rate } from 'k6/metrics'

const errors = new Rate('errors')

export const options = {
    scenarios: {
        constant_load: {
            executor: 'constant-vus',
            vus: 100,                // virtual user
            duration: '30s'
        },
        spike: {
            executor: 'ramping-vus',
            startVUs: 0,
            stages: [
                { duration: '30s', target: 100 },
                { duration: '1m',  target: 1000 },
                { duration: '30s', target: 0 }
            ]
        }
    },
    thresholds: {
        http_req_duration: ['p(95)<500', 'p(99)<2000'],
        errors: ['rate<0.01']
    }
}

export default function () {
    const res = http.post('https://api.example.com/users',
        JSON.stringify({ name: 'test' }),
        { headers: { 'Content-Type': 'application/json' } }
    )

    const ok = check(res, {
        'status is 200': r => r.status === 200,
        'response < 500ms': r => r.timings.duration < 500
    })

    errors.add(!ok)
    sleep(1)
}
```

```bash
k6 run script.js

# cloud
k6 cloud script.js

# influxdb 로 send (Grafana)
k6 run --out influxdb=http://influxdb:8086/k6 script.js
```

→ JS 익숙. modern threshold / metric.

---

## 5. Locust (Python)

```python
# locustfile.py
from locust import HttpUser, task, between

class WebsiteUser(HttpUser):
    wait_time = between(1, 3)

    def on_start(self):
        self.token = self.client.post("/auth", json={
            "user": "test", "pass": "secret"
        }).json()["token"]

    @task(3)
    def view_product(self):
        self.client.get("/api/products/123", headers={"Auth": self.token})

    @task(1)
    def buy(self):
        self.client.post("/api/orders", json={"product": 123}, headers={"Auth": self.token})
```

```bash
locust -f locustfile.py --host=https://api.example.com -u 1000 -r 100 -t 5m --headless

# UI mode
locust -f locustfile.py --host=https://api.example.com
# → http://localhost:8089
```

→ Python team / 복잡 시나리오.

---

## 6. distributed load (k8s)

```yaml
# k6 operator
apiVersion: k6.io/v1alpha1
kind: TestRun
metadata: {name: load-test}
spec:
  parallelism: 4              # 4 worker pod
  script:
    configMap: {name: k6-test, file: script.js}
```

→ 한 worker 부족 시 k8s 의 N pod 로 분산.

---

## 7. scenario 디자인

```
시나리오:
  1. anonymous 70%      browse / search
  2. logged-in 20%      add to cart / checkout
  3. admin 5%           dashboard
  4. crawler bot 5%     RSS / sitemap

→ 실제 traffic pattern 반영.
```

---

## 8. 검증 기준 (threshold)

```javascript
thresholds: {
    'http_req_duration{type:api}': ['p(95)<300', 'p(99)<1000'],
    'http_req_failed': ['rate<0.01'],     // < 1% error
    'http_reqs': ['rate>100']              // > 100 RPS
}
```

→ SLO 와 동일 기준. test 가 fail = deploy block.

---

## 9. infra 준비

```
load test 의 함정:
  - load generator 가 client bottleneck
  - 같은 region (latency 영향)
  - 충분한 source IP (TIME_WAIT)
  - ephemeral port (65k)
  - NAT 의 한계
  - DNS rate limit

→ test machine 의 ulimit -n / port range 설정
→ 충분한 instance (k6 cloud 또는 distributed)
```

---

## 10. ramp-up (★)

```
0 → peak → sustained → 0

이유:
  cold start (JIT warm-up, cache warm)
  connection pool 점진 채움
  HPA 가 따라잡기

❌ 0 → 1000 즉시 → unrealistic
✓ 5min ramp → 30min sustained → 5min ramp down
```

---

## 11. 무엇을 측정

```
client side:
  - RPS (request per second)
  - latency p50/p95/p99/max
  - error rate
  - throughput (bytes/s)

server side:
  - CPU / memory / IO / network
  - thread / connection pool 사용
  - GC pause / count
  - DB query 시간
  - cache hit rate
  - upstream API latency

→ 둘 다 본 후 결론.
```

---

## 12. soak test (24h+)

```javascript
export const options = {
    scenarios: {
        endurance: {
            executor: 'constant-vus',
            vus: 200,
            duration: '24h'    // 1 day
        }
    }
}
```

찾는 것:
- memory leak (RSS 점점 증가)
- file descriptor leak
- thread accumulation
- DB connection leak
- log file 폭증

---

## 13. spike test

```javascript
stages: [
    { duration: '1m',  target: 100 },     // normal
    { duration: '10s', target: 5000 },    // spike!
    { duration: '3m',  target: 5000 },    // sustain
    { duration: '10s', target: 100 },     // drop
    { duration: '3m',  target: 100 }      // recovery
]
```

찾는 것:
- circuit break 동작
- HPA 따라잡기
- thundering herd (cache miss 동시)
- 다 떨어진 후 recovery time

---

## 14. CI / production-like

```
staging:
  prod 와 동일 instance type / config / data 크기
  
test 환경 OK 도 → prod 영향 다를 수 있음
→ chaos + load 결합 (production 의 1%)
```

---

## 15. 함정

1. **dev 의 load** — prod 와 다름, 의미 X.
2. **load generator 가 bottleneck** — distributed.
3. **single user / single endpoint** — 비현실.
4. **ramp 없음** — cold start 영향.
5. **threshold 없음** — pass/fail 불명확.
6. **server-side metric 안 봄** — client 만.
7. **soak 안 함** — leak miss.
8. **production 직접 load test** — 동의 없이 X.

---

## 16. 관련

- [[performance|↑ performance]]
- [[capacity-planning]]
- [[../monitoring/slos-and-sli|↗ SLO]]
- [[../cicd/pipeline-patterns|↗ CI/CD]]
