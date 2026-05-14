---
title: "추천 (넷플릭스·왓챠 스타일) — viewing history / collab filter"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - recommendation
  - personalization
---

# 추천 (넷플릭스·왓챠 스타일) — viewing history / collab filter

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | viewing/event 적재 / 협업필터 기초 / cold start |

**[[api-design|↑ api-design hub]]**

> 📐 공통: [[../common/response-envelope]]. 본 레시피는 **백엔드 통합 위주** — 모델 학습은 ML 팀.

---

## 1. 추천 시스템 — 백엔드의 역할

```
[사용자 행동]
  view / click / like / purchase / dwell time
       ↓
[이벤트 적재]
  Kafka producer (실시간) + S3/data lake (배치)
       ↓
[모델 학습 — ML 팀]
  Spark / SageMaker / 추천 모델
       ↓
[추천 결과 저장]
  Redis ZSET (user_id → ranked item ids) 또는 DynamoDB
       ↓
[백엔드 추천 API]
  GET /recommendations/for-you  →  Redis 조회 + 메타 hydrate
```

→ 백엔드 = 이벤트 적재 + 추천 결과 서빙. 모델은 별도 시스템.

---

## 2. 이벤트 적재

### 2.1 이벤트 종류

```java
public enum InteractionType {
    VIEW,            // 페이지 본 것
    CLICK,           // 클릭
    LIKE,
    BOOKMARK,
    ADD_TO_CART,
    PURCHASE,
    DWELL,           // 머무른 시간 (n초 이상)
    SHARE,
    REVIEW
}
```

### 2.2 endpoint

```http
POST /api/v1/events
{
  "type": "VIEW",
  "targetType": "PRODUCT",
  "targetId": "01HZ-PRODUCT...",
  "context": { "source": "search", "position": 3 }
}
```

### 2.3 Kafka producer 식

```java
@Service
@RequiredArgsConstructor
public class InteractionEventService {

    private final KafkaTemplate<String, String> kafka;
    private final ObjectMapper json;
    private final Clock clock;

    public void record(UserId userId, InteractionType type,
                       String targetType, String targetId, Map<String, Object> context) {
        var event = new InteractionEvent(
            UUID.randomUUID().toString(),
            userId == null ? null : userId.value(),
            type, targetType, targetId, context,
            Instant.now(clock)
        );
        try {
            // partition key = userId (같은 user 의 이벤트 순서 보장)
            kafka.send("interaction-events",
                userId == null ? targetId : userId.value(),
                json.writeValueAsString(event));
        } catch (Exception e) {
            log.warn("interaction event publish failed", e);
            // event 적재는 best-effort — 실패해도 비즈니스 흐름 막지 않음
        }
    }
}
```

### 2.4 Controller

```java
@RestController
@RequestMapping("/api/v1/events")
@RequiredArgsConstructor
public class EventCollectorController {

    private final InteractionEventService service;

    @PostMapping
    public ResponseEntity<CommonResponse<Void>> record(
        @Valid @RequestBody RecordEventRequest req,
        @AuthenticationPrincipal User user
    ) {
        var userId = user == null ? null : user.id();
        service.record(userId, req.type(), req.targetType(), req.targetId(), req.context());
        return ResponseEntity.accepted().body(CommonResponse.success(ResponseCode.OK, "이벤트 적재"));
    }
}

public record RecordEventRequest(
    @NotNull InteractionType type,
    @NotBlank String targetType,
    @NotBlank String targetId,
    Map<String, Object> context
) {}
```

→ `202 Accepted` 응답 — 적재는 best-effort. 클라가 응답 안 기다리도록 fire-and-forget.

---

## 3. 시청기록 (Viewing History) — DB 영속

```sql
CREATE TABLE user_views (
    user_id      CHAR(26) NOT NULL,
    target_type  VARCHAR(30) NOT NULL,           -- PRODUCT / VIDEO / POST
    target_id    CHAR(26) NOT NULL,
    last_viewed_at TIMESTAMPTZ NOT NULL,
    view_count   INTEGER NOT NULL DEFAULT 1,
    PRIMARY KEY (user_id, target_type, target_id)
);
CREATE INDEX ix_user_views_user_time ON user_views (user_id, last_viewed_at DESC);

-- 동영상은 추가 — 진행률
CREATE TABLE video_progress (
    user_id     CHAR(26) NOT NULL,
    video_id    CHAR(26) NOT NULL,
    position_seconds INTEGER NOT NULL,
    duration_seconds INTEGER NOT NULL,
    completed   BOOLEAN NOT NULL DEFAULT false,
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, video_id)
);
```

```sql
-- upsert (시청 시마다)
INSERT INTO user_views (user_id, target_type, target_id, last_viewed_at, view_count)
VALUES (:userId, :type, :id, now(), 1)
ON CONFLICT (user_id, target_type, target_id)
DO UPDATE SET
    last_viewed_at = EXCLUDED.last_viewed_at,
    view_count = user_views.view_count + 1;
```

---

## 4. 추천 결과 조회

### 4.1 결과는 Redis ZSET — ML 팀이 채움

```
Key: rec:user:{userId}:home
Members: item ids
Scores: 추천 점수 (높을수록 우선)
TTL: 1시간 (매시 새로 계산)
```

### 4.2 백엔드 — Redis 에서 ID 가져와 메타 hydrate

```java
@Service
@RequiredArgsConstructor
public class RecommendationService {

    private final RedisTemplate<String, String> redis;
    private final ProductRepository products;
    private final TrendingFallback fallback;

    public List<RecommendedItem> forUser(UserId userId, RecKind kind, int limit) {
        var key = "rec:user:" + userId.value() + ":" + kind.name().toLowerCase();
        var typedTuples = redis.opsForZSet().reverseRangeWithScores(key, 0, limit - 1);

        if (typedTuples == null || typedTuples.isEmpty()) {
            // cold start — fallback (인기 상품 / 신상)
            return fallback.fallback(userId, kind, limit);
        }

        var ids = typedTuples.stream()
            .map(t -> new ProductId(t.getValue()))
            .toList();

        // 메타 hydrate — 한 쿼리
        var found = products.findAllByIds(ids);
        var byId = found.stream().collect(Collectors.toMap(p -> p.id().value(), p -> p));
        return typedTuples.stream()
            .map(t -> {
                var p = byId.get(t.getValue());
                if (p == null) return null;          // 삭제된 상품
                return new RecommendedItem(p, t.getScore());
            })
            .filter(Objects::nonNull)
            .toList();
    }
}

public enum RecKind { HOME, SIMILAR, BECAUSE_YOU_VIEWED }
```

### 4.3 추천 API

```java
@GetMapping("/api/v1/recommendations/for-you")
@PreAuthorize("isAuthenticated()")
public ResponseEntity<CommonResponse<List<RecommendedItem>>> forYou(
    @RequestParam(defaultValue = "HOME") RecKind kind,
    @RequestParam(defaultValue = "20") @Min(1) @Max(50) int limit,
    Authentication auth
) {
    return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
        rec.forUser(new UserId(auth.getName()), kind, limit), "추천"));
}

@GetMapping("/api/v1/products/{id}/similar")
public ResponseEntity<CommonResponse<List<RecommendedItem>>> similar(
    @PathVariable String id, @RequestParam(defaultValue = "10") int limit
) {
    return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
        rec.similar(new ProductId(id), limit), "유사 상품"));
}
```

---

## 5. Cold Start — 추천 결과 없을 때

신규 user / 추천 모델 미구축 시 fallback:

```java
@Service
@RequiredArgsConstructor
public class TrendingFallback {

    private final ProductStatsRepository stats;
    private final UserViewRepository views;

    public List<RecommendedItem> fallback(UserId userId, RecKind kind, int limit) {
        return switch (kind) {
            case HOME -> {
                // 1순위: 최근 본 카테고리의 인기 상품
                var recentCategories = views.recentCategoriesFor(userId, 5);
                if (!recentCategories.isEmpty()) {
                    yield stats.topByCategories(recentCategories, limit);
                }
                // 2순위: 전체 인기
                yield stats.topProducts30d(limit);
            }
            case SIMILAR, BECAUSE_YOU_VIEWED -> List.of();   // 못 만듦
        };
    }
}
```

---

## 6. 간단한 협업 필터 (DB 기반) — ML 팀 전인 작은 서비스

복잡한 추천 시스템 없이 SQL 만으로 "이걸 산 사람이 같이 산 상품":

```sql
-- 같은 주문에 들어간 상품 쌍의 빈도
WITH co AS (
    SELECT
        a.product_id AS source,
        b.product_id AS target,
        count(*) AS cnt
    FROM order_items a
    JOIN order_items b ON a.order_id = b.order_id AND a.product_id <> b.product_id
    JOIN orders o ON o.id = a.order_id
    WHERE o.status IN ('PAID', 'SHIPPED', 'DELIVERED')
      AND o.created_at > now() - INTERVAL '90 days'
    GROUP BY a.product_id, b.product_id
)
SELECT target AS recommended_id, cnt
FROM co
WHERE source = :productId
ORDER BY cnt DESC LIMIT 10;
```

→ Materialized View 로 일 1회 갱신. 단순 + 효과적.

---

## 7. Spring Batch — 야간 갱신 job (옵션)

```java
@Configuration
@RequiredArgsConstructor
public class RecomputeRecommendationsJob {

    private final RedisTemplate<String, String> redis;
    private final UserViewRepository views;
    private final ProductCoOccurrenceRepository cooc;

    @Scheduled(cron = "0 0 4 * * *")   // 매일 새벽 4시
    @SchedulerLock(name = "recomputeRec", lockAtMostFor = "PT3H")
    public void run() {
        var activeUserIds = views.activeUserIdsInLastDays(7, 10000);
        for (var uid : activeUserIds) {
            var topItems = computeFor(uid);
            var key = "rec:user:" + uid + ":home";
            redis.delete(key);
            topItems.forEach(item -> redis.opsForZSet().add(key, item.id(), item.score()));
            redis.expire(key, Duration.ofHours(25));   // 다음 갱신까지
        }
    }

    private List<ScoredItem> computeFor(String userId) {
        // 사용자 view history → 동시구매 매트릭스 → 점수
        // 본 레시피 범위 밖. ML 또는 별도 알고리즘 노트.
        return List.of();
    }
}
```

---

## 8. 함정 모음

### 함정 1 — 추천 결과가 stale
TTL 만료 안 됐는데 사용자가 신상 본 직후 추천에 안 뜸. **실시간 신호 + 배치 결합** — Redis 의 last_action 도 점수 반영.

### 함정 2 — 모든 view 를 DB upsert
hot user 가 분당 100 view = DB 부하. Kafka → batch upsert (5분).

### 함정 3 — 추천 ID 가 삭제된 상품
Hydrate 시 null → 응답에서 필터링. Redis 에서도 정기 정리.

### 함정 4 — Cold start 무대응
신규 user 응답 빈 list. **fallback (인기 / 신상)** 필수.

### 함정 5 — `ZADD` 의 score 가 너무 클 / 작
JS number safe (2^53) 범위. 추천 점수 0~1 또는 0~100 정규화.

### 함정 6 — 추천 다양성 0
같은 카테고리 / 같은 브랜드만 반복. **MMR** (Maximal Marginal Relevance) 또는 diversification.

### 함정 7 — 개인 추천에 PII 노출
"홍길동이 본 상품" 처럼 식별. UX 의도여도 사용자 동의 필요.

### 함정 8 — 인기 / 트렌딩과 개인 추천 혼동
유저별 ZSET vs 전체 ZSET. key prefix 명확.

### 함정 9 — Kafka 다운 시 비즈니스 차단
event publish 가 동기로 비즈니스 흐름 막음. **try-catch + log**, 최악엔 적재 누락 감수.

### 함정 10 — A/B test 없이 모델 교체
새 모델 = 매출 폭락 가능. **canary + 매출 비교**.

---

## 9. 운영 체크리스트

- [ ] event publish 는 best-effort (실패 OK)
- [ ] Kafka partition key = userId
- [ ] Redis ZSET TTL (모델 주기와 동기)
- [ ] cold start fallback (인기 상품)
- [ ] hydrate 시 stale ID 필터
- [ ] 추천 클릭율 (CTR) 모니터 — A/B test
- [ ] PII 노출 검토
- [ ] ZSET 정기 정리 (만료된 결과)

---

## 10. 관련

- [[product-search]] — 기본 search 와 통합
- [[feed-timeline]] — feed 도 추천 결합
- [[cache-redis]] — 추천 결과 캐싱
- 이벤트 / Kafka (예정)
- [[api-design|↑ api-design hub]]
