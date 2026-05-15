---
title: "Feature flags — LaunchDarkly / Flagsmith / Unleash"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:32:00+09:00
tags: [devops, cicd, feature-flags]
---

# Feature flags — LaunchDarkly / Flagsmith / Unleash

**[[cicd|↑ cicd]]**

---

## 1. 왜 (★ trunk-based 의 필수)

```
deploy ≠ release:
  코드 merge / deploy → 사용자에게 안 보임
  flag 으로 점진 release

용도:
  - A/B test
  - canary
  - dark launch
  - kill switch (긴급 off)
  - permission (premium 만)
  - geo (region 별 다른 feature)
  - team / user / cohort 별
```

---

## 2. 도구

| | type | 강점 |
| --- | --- | --- |
| **LaunchDarkly** (★) | SaaS | 표준, mature, multi-language |
| **Flagsmith** | OSS + SaaS | 자체 host 가능 |
| **Unleash** | OSS + SaaS | OSS first |
| **GrowthBook** | OSS + SaaS | experimentation 강 |
| **Split.io** | SaaS | A/B test 강 |
| **ConfigCat** | SaaS | 가격 친화 |
| **PostHog** | OSS + SaaS | product analytics + flags |
| **자체 구현** | DB / Redis | 단순 case |

→ enterprise = LaunchDarkly. OSS = Flagsmith / Unleash.

---

## 3. flag 종류

```
release flag:
  새 feature 의 점진 release
  단기 사용 후 제거

experiment flag:
  A/B test
  metric 별 차이 측정

ops flag:
  kill switch / circuit
  영구 (없으면 위험)

permission flag:
  premium / beta / employee 만
  영구
```

---

## 4. 사용 예 (LaunchDarkly Java)

```java
LDClient client = new LDClient("sdk-key");

// boolean flag
LDUser user = new LDUser.Builder("user-id-123")
    .email("alice@example.com")
    .custom("tier", "premium")
    .build();

if (client.boolVariation("new-checkout", user, false)) {
    return newCheckout();
} else {
    return oldCheckout();
}

// string variant
String theme = client.stringVariation("ui-theme", user, "default");

// numeric (A/B)
int discount = client.intVariation("discount-percent", user, 0);

// 종료
client.close();
```

---

## 5. server-side vs client-side

```
server-side:
  - 모든 사용자 evaluation 서버에서
  - secret SDK key
  - 안전 (client 가 flag 못 봄)
  - latency

client-side (Browser / mobile):
  - SDK 가 client 에 evaluation
  - public client-id
  - 빠름 (no roundtrip)
  - flag rule 노출 가능 (security 검토)
```

---

## 6. targeting rule

```
new-checkout flag:
  
  Rule 1: tier == "premium" → ON
  Rule 2: country == "KR" → 50% ON
  Rule 3: user_id ends with "abc" → ON (internal team)
  Default: OFF

→ UI 에서 정의. 코드 변경 X.
```

```yaml
# Unleash strategy
strategies:
  - name: gradualRolloutUserId
    parameters:
      percentage: "50"
      groupId: new-checkout
  - name: userWithId
    parameters:
      userIds: "internal-team-1,internal-team-2"
```

---

## 7. progressive delivery 패턴

```
Day 1: internal user 만 (employees)
       관찰 → bug 없음
       
Day 2: beta (1%)
       관찰 → metric OK
       
Day 3: 5%
Day 4: 25%
Day 5: 50%
Day 6: 100%

Week 2: flag cleanup (코드 제거)
```

각 단계:
- error rate / latency 모니터
- user feedback
- rollback button (즉시 0%)

---

## 8. A/B test (★)

```
flag = 3 variant:
  control (50%)
  variantA (25%): "Buy" button
  variantB (25%): "Add to cart" button

metric:
  conversion rate
  revenue per user

도구 = analytics 통합:
  - LaunchDarkly Experimentation
  - GrowthBook
  - Optimizely
  - Statsig
  - Eppo

→ statistical significance 도구가 자동 계산.
```

---

## 9. flag debt (★)

```
시간 지나면 flag 가 누적:
  - 사용 안 하는 flag
  - 100% on 인데 안 cleanup
  - 더 이상 의미 없는 code path

해결:
  - 만료 날짜 (LaunchDarkly 의 stale flag)
  - quarterly cleanup
  - lint rule (code 의 flag 이름 vs LD 의 list)
  - PR 의 flag introduction = cleanup 약속
```

→ flag debt = tech debt.

---

## 10. 자체 구현 (단순 case)

```sql
CREATE TABLE feature_flags (
    name VARCHAR(100) PRIMARY KEY,
    enabled BOOLEAN DEFAULT false,
    rollout_percent INT DEFAULT 0,
    targeting JSONB DEFAULT '[]',
    updated_at TIMESTAMP
);
```

```java
// Redis cache
public boolean isEnabled(String flag, User user) {
    Flag f = cache.get(flag);
    if (f == null) f = loadFromDb(flag);
    
    if (!f.isEnabled()) return false;
    
    // user-specific
    if (f.getTargeting().contains(user.getId())) return true;
    
    // percentage
    int hash = Math.abs((flag + user.getId()).hashCode()) % 100;
    return hash < f.getRolloutPercent();
}
```

→ SaaS 도구 의 작은 subset. 단순 case 만.

---

## 11. backend + frontend 동기화

```
issue: frontend 의 cached flag vs backend 의 변경

해결:
  - 짧은 cache TTL
  - SSE / WebSocket 으로 push (LaunchDarkly)
  - server-rendered (SSR) 에서 flag 결정
```

---

## 12. metric / alerting

```
flag 별:
  - eval count
  - eval latency
  - variation 분포
  - error during eval

flag 변경:
  - 누가 / 언제 변경 (audit log)
  - on/off 직후 metric 비교
  - rollback button
```

---

## 13. test 의 flag

```java
// unit / integration 에서 flag override
LDClient.usingTestMode(true);
LDClient.setVariation("new-checkout", true);

// 또는 dependency injection
@TestConfiguration
public class TestConfig {
    @Bean
    @Primary
    public FlagService mockFlag() {
        return name -> name.equals("new-checkout");
    }
}
```

---

## 14. permission ≠ release flag

```
release flag = 임시, cleanup
permission flag = 영구

permission:
  - subscription tier
  - beta program enrollment
  - feature entitlement

→ permission 은 RBAC / entitlement service 가 더 맞음.
   flag 도구 = release / experiment 위주.
```

---

## 15. 함정

1. **flag cleanup 안 함** — debt 누적.
2. **flag 안 보고 코드 작성** — broken state.
3. **flag SDK fail 시 default** — fail-closed vs fail-open.
4. **cache 너무 김** — 변경 늦음.
5. **flag 의 secret 정보 leak** — client-side.
6. **flag 너무 많음** — 복잡도 폭주.
7. **flag log 없음** — 누가 무엇 변경했는지 모름.

---

## 16. 관련

- [[cicd|↑ cicd]]
- [[trunk-based-development]]
- [[release-strategies]]
- [[../sre/runbook|↗ runbook (kill switch)]]
