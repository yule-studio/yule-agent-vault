---
title: "Graceful degradation — 부분 fail 시 core 보존 ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:09:00+09:00
tags: [devops, sre, degradation, resilience]
---

# Graceful degradation — 부분 fail 시 core 보존 ★

**[[sre|↑ sre]]**

---

## 1. 무엇

```
한 component fail / overload:
  
naive:
  전체 fail → 사용자 모두 영향
  
graceful:
  fail 한 부분만 차단
  → core service 살리기
  → 사용자 일부 feature 만 영향
```

→ "100% 서비스" 대신 "core 100% + 부수 best effort".

---

## 2. degradation 의 spectrum

```
full functional ─────────── degraded ─────────── unavailable
   100% feature                                       0%
   
goal:
  fail 시 unavailable 가지 말고 degraded 로.
```

---

## 3. 흔한 degradation 패턴 (★)

### A. read-only mode

```
write DB fail / migration:
  → write disable
  → read 만 OK
  → "잠시 후 다시 시도" 안내
  
사용자:
  - 사이트 browse 가능
  - 결제 / 가입 X
  - 로그인 / 검색 OK
```

### B. fallback content

```
recommendation service fail:
  → 인기 상품 list
  
personalization fail:
  → default content
  
search service fail:
  → cached top result
  
profile picture CDN fail:
  → default avatar
```

### C. async processing

```
synchronous → asynchronous:
  
  notification:
    fail → queue 에 넣음
    "잠시 후 메일 보내드림"
    
  payment:
    fail → 처리 중 상태
    background retry
```

### D. cached stale

```
DB / cache miss:
  → 1 hour old data OK 인가?
  → yes → stale 응답
  → freshness 정책 별 결정
```

### E. degraded feature

```
non-critical feature off:
  - 추천
  - personalization
  - analytics tracking
  - 광고
  - chat widget
  - 실시간 update (polling 으로)
```

### F. external service replace

```
primary payment gateway fail:
  → secondary 로
  
primary email service fail:
  → SES → SendGrid
  
primary CDN fail:
  → fallback origin
```

---

## 4. 결정 매트릭스 (★)

```
feature 별 SLA / criticality:

P0 (always on):
  - login / logout
  - 결제
  - 주문 view
  
P1 (graceful):
  - 검색 (cached fallback)
  - 추천 (인기 상품)
  - cart sync (eventual)
  
P2 (degraded):
  - personalization
  - "for you" feed
  - 실시간 notification
  
P3 (drop OK):
  - analytics
  - 광고
  - tracking
  - logging (sampled)

→ 각 feature 의 SLO 다름.
   capacity fail 시 차등 우선순위.
```

---

## 5. circuit breaker + degradation (★)

```java
@CircuitBreaker(name = "recommendation", fallbackMethod = "fallback")
public List<Product> recommend(Long userId) {
    return recommendationClient.getRecommendations(userId);
}

public List<Product> fallback(Long userId, Throwable t) {
    log.warn("recommendation degraded: {}", t.getMessage());
    return popularProductService.getTopN(10);   // 인기 상품
}
```

→ [[../distributed-systems/circuit-breaker|circuit-breaker]] 와 결합.

---

## 6. feature flag 활용 (★)

```
operational feature flag:
  
  - search-personalization:on
    → 정상 = personalized search
    → off = default search (degraded)
    
  - new-recommendation:on
    → fail 시 즉시 off
    
  - real-time-feed:on
    → busy 시 polling fallback

→ runtime 으로 즉시 off.
   incident 시 즉각 action.
```

---

## 7. load shedding 과 (★)

```
overload 시:
  
  P0 = 항상 accept
  P1 = 90% capacity 까지
  P2 = 70%
  P3 = 50%
  
→ 우선순위 별 reject.
   "shedding" 도 degradation 의 일종.

→ [[load-shedding]] 참조.
```

---

## 8. user 체험 (★)

```
사용자에게 보이는 것:

bad:
  - 500 error
  - timeout (loading 만)
  - 빈 페이지
  - 무한 spinning

good:
  - "추천 system 일시 오류, 인기 상품을 보여드립니다" 안내
  - skeleton UI
  - 기본 content 로 채움
  - "잠시 후 다시 시도해 주세요"
  - 알림 (email 으로 처리 완료)

→ "기능 제한" 명시 + 다음 action 안내.
```

---

## 9. 실전 예 — 큰 사이트

```
Amazon's "everything store" 의 degradation:
  
  - recommendation fail → 인기 상품
  - search fail → cached top
  - personalization fail → default
  - review service fail → review 안 보임 (구매 가능)
  - inventory 정확도 ↓ → 일시 표시 (결제 시 재확인)
  - 가격 update 지연 → cached 가격
  
→ 결제 / 주문 만 100% 정확.
   나머지 = best effort.
```

---

## 10. design 원칙

```
1. dependency 의 criticality 분류
   - 없으면 fail vs 없으면 약간 limited

2. timeout / circuit / fallback 설정
   - 모든 외부 call

3. UI 의 graceful 표시
   - 빈 chunk OK
   - skeleton

4. monitoring 의 health 분리
   - core vs non-core

5. test
   - chaos engineering
   - dependency fail 시뮬

6. runbook
   - 의도적 degradation 절차
```

---

## 11. observability

```
metric:
  - feature flag state (on/off)
  - fallback rate (예: recommendation fallback)
  - degraded mode entry / exit
  - per-feature error rate
  - user impact (degraded user %)

alert:
  - fallback rate > 10%
  - degraded mode 가 1h+ 지속
  - core feature 영향 (P0 fail)
```

---

## 12. progressive enhancement

```
디자인 원칙:
  
  base layer: 가장 단순 / 안정 (HTML)
    → 모든 client / 모든 상황
  
  enhancement layer:
    JS / CSS / 동적 feature
    → 가능하면 OK, 안 되면 fallback
  
  premium layer:
    real-time / personalized
    → 잘 동작하면 좋음, 아니면 base
```

→ frontend / backend 의 동일 사상.

---

## 13. testing

```
chaos engineering:
  - dependency kill
  - latency 주입
  - error 강제
  - traffic spike
  
도구:
  - Chaos Mesh (k8s)
  - LitmusChaos
  - Gremlin
  - AWS Fault Injection

테스트 시나리오:
  - DB master down → read-only mode?
  - Redis down → cache miss handling?
  - external API timeout → fallback?
  - 50% pod down → load balanced?
```

---

## 14. 함정

1. **all-or-nothing 디자인** — degradation impossible.
2. **fallback 의 fail** — nested fallback 필요.
3. **degradation 의 monitoring 없음** — 사용자 영향 모름.
4. **UI 가 hardcoded** — backend 의 partial response 처리 X.
5. **operational flag 없음** — runtime degradation 불가.
6. **test 없음** — 실제 fail 시 처음.
7. **degraded mode 가 영원** — recovery automation 없음.
8. **dependency criticality 미정의** — 모두 P0.

---

## 15. 관련

- [[sre|↑ sre]]
- [[load-shedding]]
- [[../distributed-systems/circuit-breaker|↗ circuit breaker]]
- [[../cicd/feature-flags-integration|↗ feature flag]]
- [[runbook]]
