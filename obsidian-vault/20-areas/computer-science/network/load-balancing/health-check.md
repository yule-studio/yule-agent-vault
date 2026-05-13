---
title: "Health Check — 백엔드 상태 점검"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T06:20:00+09:00
tags:
  - network
  - load-balancing
  - health-check
---

# Health Check — 백엔드 상태 점검

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Active / Passive / Outlier |

**[[load-balancing|↑ Load Balancing]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

LB 가 **backend 의 정상 여부** 를 판단해 트래픽 분산 결정.

---

## 2. Active vs Passive

### Active Health Check
```
LB → Backend: GET /health
Backend → LB: 200 OK
        ↓
LB 가 backend "healthy" 표시
```

- 주기적 probe (예: 5 초마다)
- 응답 200 / 200 미만 임계 → unhealthy
- 명확한 신호

### Passive Health Check (Outlier Detection)
```
실제 트래픽 응답 관찰:
- 5xx 비율 ↑ → unhealthy
- Latency ↑ → unhealthy
- Timeout ↑ → unhealthy
```

- Probe 없음 — 실 트래픽 기반
- Envoy / Linkerd / Istio

### 둘 다 권장
- Active — 명시적 / 빠른 detect
- Passive — 실제 부하 반영

---

## 3. 프로토콜 별 Health Check

### TCP (L4)
```
LB → Backend: TCP SYN
Backend → LB: SYN-ACK
LB → Backend: RST (즉시 끊음)
```

→ Application 가 down 인지 모름 (port 만 열림 OK).

### HTTP (L7)
```
GET /health HTTP/1.1
Host: backend
        ↓
200 OK + JSON 상태
```

### HTTP/2 / gRPC
- gRPC health checking protocol (RFC draft)
- `grpc.health.v1.Health/Check`

### TLS
- handshake 만 점검 (가벼움)

### Custom
- SQL ping
- Redis PING
- Memcached version

---

## 4. /health endpoint — 구현

### Liveness vs Readiness (Kubernetes)

| 종류 | 의미 |
| --- | --- |
| **Liveness** | 프로세스가 살아있나? → 죽었으면 재시작 |
| **Readiness** | 트래픽 받을 준비? → 안 되면 LB 제외 |
| **Startup** | 시작 완료? → 시작 중 liveness 비활성 |

### Liveness
```python
@app.route('/healthz')
def liveness():
    return 'OK', 200
```

→ 매우 단순 — 프로세스 응답.

### Readiness
```python
@app.route('/ready')
def readiness():
    if not db.connected():
        return 'DB down', 503
    if not redis.connected():
        return 'Redis down', 503
    if cpu_load > 0.95:
        return 'Overload', 503
    return 'Ready', 200
```

→ 의존성 / 부하 확인.

### Deep / Shallow
- **Shallow** — 자기 자신만 (process)
- **Deep** — 의존성도 (DB / Cache) — 위험 (cascading failure)

---

## 5. 임계 / 정책

### Threshold
```
healthy_threshold = 2     ; 연속 2 번 OK → healthy
unhealthy_threshold = 3   ; 연속 3 번 fail → unhealthy
interval = 5s             ; 5 초마다 probe
timeout = 2s              ; 2 초 안에 응답
```

### Drain (Graceful)
```
Backend healthy → unhealthy:
1. 새 connection 전송 X
2. 기존 connection — 완료까지 (drain timeout 30s)
3. 영구 제거
```

---

## 6. AWS Target Group Health Check

```
Protocol: HTTP / HTTPS
Path: /health
Port: traffic-port
Healthy threshold: 5
Unhealthy threshold: 2
Timeout: 5 s
Interval: 30 s
Matcher: 200-399
```

---

## 7. Outlier Detection (Envoy)

### 정의
- Passive — backend 의 행동 분석
- 자동 ejection / re-add

### 트리거
- consecutive_5xx
- consecutive_gateway_failure
- success_rate (statistical outlier)

### 설정
```yaml
outlier_detection:
  consecutive_5xx: 5
  interval: 10s
  base_ejection_time: 30s
  max_ejection_percent: 50
```

→ 5 번 5xx → 30 초 ejection.

### 효과
- Passive — probe 부담 X
- 실제 부하 기반

---

## 8. Cascading Failure 방지

### 문제
- /health 가 DB 도 확인 → DB down → 모든 backend unhealthy → 모두 LB 제외 → 503

### 해결
- /health (liveness) — 의존성 X
- /ready (readiness) — 의존성 O, but partial degradation 허용
- Circuit breaker — 의존성 실패 시 fallback

---

## 9. Health check 의 부하

### Probe 부담
```
backend N 개, LB M 개, interval I 초
→ M × N × (1/I) probe per second
```

### 큰 cluster
- N = 1000, M = 10, I = 5 → 2000 probe/s

### 최적화
- Probe 가벼움 (단순 GET)
- 캐시 가능
- Passive 와 결합 (probe 빈도 ↓)

---

## 10. Kubernetes — Probe

### YAML
```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 2
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
      failureThreshold: 2
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
```

### 차이
- **Liveness fail** → pod restart
- **Readiness fail** → service 에서 제외 (pod 살아있음)
- **Startup fail** → liveness 재시작 (시작 중)

---

## 11. Synthetic Monitoring

### 정의
- 외부 / 사용자 관점 health
- Datadog Synthetics, Pingdom, Uptrends

### 특징
- 사용자 흐름 (login → action) 모방
- 다중 region 에서 probe
- Alert / SLA

---

## 12. 함정

### 함정 1 — /health 가 DB 확인
DB down → 모든 backend unhealthy → 503. Liveness 와 readiness 분리.

### 함정 2 — Probe 너무 자주
부하 ↑. Interval 5-30s.

### 함정 3 — Probe 너무 가벼움
Application 의 실제 문제 못 잡음. /ready 가 의미 있게.

### 함정 4 — Timeout 짧음
GC / 일시 slow → false unhealthy. Threshold + timeout 조정.

### 함정 5 — Drain 없는 deploy
Connection 끊김. Graceful shutdown / preStop hook.

### 함정 6 — Cascading failure
모든 backend 의존성 — 한 실패 → 전체. Bulkhead / circuit breaker.

### 함정 7 — Probe 의 신뢰 검증
LB 의 source IP 만 허용 — 외부의 /health 노출 X.

---

## 13. 학습 자료

- Envoy Outlier Detection docs
- Kubernetes Probes docs
- "Release It!" (Nygard) — Stability patterns
- Google SRE Book — Health checking

---

## 14. 관련

- [[load-balancing]] — Hub
- [[load-balancing-algorithms]] — Health 기반 선택
- [[l4-load-balancing]] / [[l7-load-balancing]]
