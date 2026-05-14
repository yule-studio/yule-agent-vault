---
title: "리뷰 / 평점 (무신사·쿠팡 스타일) — Java Spring Boot"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - review
  - rating
---

# 리뷰 / 평점 (무신사·쿠팡 스타일) — Java Spring Boot

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 1인1리뷰 / 이미지 / 평점 집계 / 도움돼요 / moderation |

**[[api-design|↑ api-design hub]]**

> 📐 공통: [[../common/response-envelope]] · [[../common/security-config]]. 전제: [[product-crud]] / [[order-stock]].

---

## 1. 무엇을 만드는가

```
POST   /api/v1/products/{id}/reviews           # 리뷰 작성 (구매자만, 1인1리뷰)
GET    /api/v1/products/{id}/reviews?sort=...  # 상품 리뷰 목록
PATCH  /api/v1/reviews/{id}                    # 내 리뷰 수정 (작성 후 30일)
DELETE /api/v1/reviews/{id}                    # 내 리뷰 삭제 (soft)
POST   /api/v1/reviews/{id}/helpful            # 도움돼요 토글
POST   /api/v1/reviews/{id}/report             # 신고
GET    /api/v1/users/me/reviews                # 내가 쓴 리뷰
```

### 1.1 비기능

- **구매 확인** — 결제 완료된 주문의 상품만 리뷰 가능
- **1인 1리뷰** — `(user_id, product_id)` UNIQUE
- **이미지 0~5장** (S3 presigned)
- **평점 1~5**
- **평점 집계** = 비정규화 (`product_stats.review_avg`, `review_count`) — 캐시
- **moderation** — 신고 N회 = hidden + 검토 큐
- **도움돼요** 카운트 — Redis hot key 방어

---

## 2. 도메인 / DB

```sql
CREATE TABLE reviews (
    id              CHAR(26) PRIMARY KEY,
    user_id         CHAR(26) NOT NULL REFERENCES users(id),
    product_id      CHAR(26) NOT NULL REFERENCES products(id),
    order_item_id   CHAR(26) NOT NULL REFERENCES order_items(id),    -- 구매 확인 + 1인1리뷰 보장
    rating          SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    content         TEXT NOT NULL,
    helpful_count   INTEGER NOT NULL DEFAULT 0,
    report_count    INTEGER NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'PUBLISHED',
        -- PUBLISHED / HIDDEN_BY_REPORT / DELETED
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_reviews_user_product ON reviews (user_id, product_id);
CREATE INDEX ix_reviews_product_status ON reviews (product_id, status, created_at DESC);
CREATE INDEX ix_reviews_user ON reviews (user_id, created_at DESC);

CREATE TABLE review_images (
    id            CHAR(26) PRIMARY KEY,
    review_id     CHAR(26) NOT NULL REFERENCES reviews(id) ON DELETE CASCADE,
    image_url     VARCHAR(500) NOT NULL,
    sort          INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX ix_review_images_review ON review_images (review_id, sort);

CREATE TABLE review_helpfuls (             -- 누가 도움돼요 눌렀나
    review_id   CHAR(26) NOT NULL REFERENCES reviews(id),
    user_id     CHAR(26) NOT NULL REFERENCES users(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (review_id, user_id)
);

CREATE TABLE review_reports (
    id          CHAR(26) PRIMARY KEY,
    review_id   CHAR(26) NOT NULL REFERENCES reviews(id),
    reporter_id CHAR(26) NOT NULL REFERENCES users(id),
    reason      VARCHAR(50) NOT NULL,
    detail      TEXT,
    status      VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_review_reports_user
    ON review_reports (review_id, reporter_id);
```

핵심:
- `(user_id, product_id) UNIQUE` — DB 가 1인1리뷰 보장
- `order_item_id` 참조 — 구매 확인 (해당 order_item 의 order.status = PAID/SHIPPED/DELIVERED 만)
- `report_count` 만 row 에, 상세는 `review_reports` 별도

---

## 3. UseCase

### 3.1 CreateReviewUseCase

```java
@Service
@RequiredArgsConstructor
public class CreateReviewUseCase {

    private final ReviewRepository reviews;
    private final OrderItemRepository orderItems;
    private final ProductStatsRepository stats;
    private final IdGenerator ids;
    private final Clock clock;
    private final ApplicationEventPublisher events;

    @Transactional
    public ReviewId handle(UserId userId, ProductId productId, OrderItemId orderItemId,
                           int rating, String content, List<String> imageUrls) {
        // 1. 구매 확인
        var orderItem = orderItems.findById(orderItemId)
            .orElseThrow(() -> new BusinessException(ResponseCode.NOT_FOUND, "주문 항목"));
        if (!orderItem.belongsToBuyer(userId))
            throw new BusinessException(ResponseCode.FORBIDDEN, "본인 주문만 리뷰 가능");
        if (!orderItem.productId().equals(productId))
            throw new BusinessException(ResponseCode.BAD_REQUEST, "주문한 상품이 아닙니다.");
        if (!orderItem.isReviewable())
            throw new BusinessException(ResponseCode.BAD_REQUEST,
                "리뷰 가능 상태가 아닙니다 (배송 완료 후).");

        // 2. 1인1리뷰 — DB UNIQUE 가 진실. 친절한 에러 위해 사전 검사도.
        if (reviews.existsByUserAndProduct(userId, productId))
            throw new BusinessException(ResponseCode.DUPLICATE_DATA, "이미 리뷰를 작성했습니다.");

        if (rating < 1 || rating > 5)
            throw new BusinessException(ResponseCode.BAD_REQUEST, "평점은 1~5");
        if (content == null || content.isBlank() || content.length() > 2000)
            throw new BusinessException(ResponseCode.BAD_REQUEST, "리뷰 내용 1~2000자");
        if (imageUrls != null && imageUrls.size() > 5)
            throw new BusinessException(ResponseCode.BAD_REQUEST, "이미지 최대 5장");

        var review = Review.create(new ReviewId(ids.next()),
            userId, productId, orderItemId, rating, content,
            imageUrls == null ? List.of() : imageUrls, Instant.now(clock));

        try {
            reviews.save(review);
        } catch (DataIntegrityViolationException e) {
            throw new BusinessException(ResponseCode.DUPLICATE_DATA, "이미 리뷰를 작성했습니다.");
        }

        // 3. 평점 집계 업데이트 (도메인 이벤트로 분리도 가능)
        events.publishEvent(new ReviewCreated(review.id(), productId, rating, Instant.now(clock)));

        return review.id();
    }
}

public record ReviewCreated(ReviewId reviewId, ProductId productId, int rating, Instant occurredAt)
    implements DomainEvent {}
```

### 3.2 평점 집계 listener

```java
@Component
@RequiredArgsConstructor
public class ProductStatsRecalculator {

    private final ProductStatsRepository stats;
    private final ReviewRepository reviews;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async
    public void on(ReviewCreated event) {
        // 단순 — 전체 평균 재계산
        var aggregate = reviews.aggregateForProduct(event.productId());
        stats.upsert(event.productId(), aggregate.count(), aggregate.average());
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async
    public void on(ReviewDeleted event) {
        var aggregate = reviews.aggregateForProduct(event.productId());
        stats.upsert(event.productId(), aggregate.count(), aggregate.average());
    }
}
```

> **함정**: 매 리뷰마다 전체 재계산 = O(N). 인기 상품 (10만+ 리뷰) 에선 느림. **증분 update** (`SET review_avg = (review_avg*review_count + :newRating) / (review_count+1)`) 또는 **batch 1분마다**.

### 3.3 ToggleHelpfulUseCase

```java
@Service
@RequiredArgsConstructor
public class ToggleHelpfulUseCase {

    private final ReviewHelpfulRepository helpfuls;
    private final ReviewRepository reviews;
    private final Clock clock;

    @Transactional
    public boolean handle(ReviewId reviewId, UserId userId) {
        var review = reviews.findById(reviewId)
            .orElseThrow(() -> new BusinessException(ResponseCode.NOT_FOUND, "리뷰"));
        if (review.isSelfWritten(userId))
            throw new BusinessException(ResponseCode.BAD_REQUEST, "자기 리뷰에 도움돼요 불가");

        var existing = helpfuls.exists(reviewId, userId);
        if (existing) {
            helpfuls.delete(reviewId, userId);
            review.decrementHelpful();
        } else {
            helpfuls.save(reviewId, userId, Instant.now(clock));
            review.incrementHelpful();
        }
        reviews.save(review);
        return !existing;
    }
}
```

> **함정 (hot key)**: 베스트 리뷰는 도움돼요가 분 단위 폭주. UPDATE 직렬화 = lock 대기. **Redis INCR 로 카운터 + 1분마다 DB sync** 옵션.

### 3.4 ReportReviewUseCase

```java
@Service
@RequiredArgsConstructor
public class ReportReviewUseCase {

    private final ReviewReportRepository reports;
    private final ReviewRepository reviews;
    private final IdGenerator ids;
    private final Clock clock;

    private static final int AUTO_HIDE_THRESHOLD = 5;

    @Transactional
    public void handle(ReviewId reviewId, UserId reporterId, String reason, String detail) {
        var review = reviews.findById(reviewId)
            .orElseThrow(() -> new BusinessException(ResponseCode.NOT_FOUND, "리뷰"));

        try {
            reports.save(new ReviewReport(new ReviewReportId(ids.next()),
                reviewId, reporterId, reason, detail, "PENDING", Instant.now(clock)));
        } catch (DataIntegrityViolationException e) {
            throw new BusinessException(ResponseCode.DUPLICATE_DATA, "이미 신고했습니다.");
        }

        var newCount = review.incrementReportCount();
        if (newCount >= AUTO_HIDE_THRESHOLD) review.hideByReport();
        reviews.save(review);
    }
}
```

→ 5회 신고 = 자동 숨김. ADMIN 검토 큐에 들어감.

---

## 4. 정렬 / 필터

```java
public enum ReviewSort {
    LATEST,         // created_at DESC
    HIGHEST_RATED,  // rating DESC, created_at DESC
    LOWEST_RATED,
    MOST_HELPFUL    // helpful_count DESC
}
```

```xml
<!-- MyBatis 의 동적 SQL -->
<select id="search" resultMap="ReviewMap">
    SELECT r.id, r.user_id, ..., u.name AS reviewer_name
    FROM reviews r
      JOIN users u ON r.user_id = u.id
    WHERE r.product_id = #{productId}
      AND r.status = 'PUBLISHED'
      <if test="hasImage != null and hasImage">
        AND EXISTS (SELECT 1 FROM review_images WHERE review_id = r.id)
      </if>
      <if test="ratingMin != null">  AND r.rating &gt;= #{ratingMin} </if>
    <choose>
      <when test="sort == 'LATEST'">         ORDER BY r.created_at DESC, r.id DESC </when>
      <when test="sort == 'HIGHEST_RATED'">  ORDER BY r.rating DESC, r.helpful_count DESC </when>
      <when test="sort == 'MOST_HELPFUL'">   ORDER BY r.helpful_count DESC, r.created_at DESC </when>
      <otherwise>                            ORDER BY r.created_at DESC </otherwise>
    </choose>
    LIMIT #{limit} OFFSET #{offset}
</select>
```

---

## 5. Controller

```java
@Tag(name = "리뷰")
@RestController
@RequestMapping("/api/v1")
@RequiredArgsConstructor
public class ReviewController {

    private final CreateReviewUseCase create;
    private final ReviewQueryService query;
    private final ToggleHelpfulUseCase toggleHelpful;
    private final ReportReviewUseCase report;

    @PostMapping("/products/{productId}/reviews")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<CommonResponse<Map<String, String>>> create(
        @PathVariable String productId,
        @Valid @RequestBody CreateReviewRequest req, Authentication auth
    ) {
        var id = create.handle(new UserId(auth.getName()),
            new ProductId(productId), new OrderItemId(req.orderItemId()),
            req.rating(), req.content(), req.imageUrls());
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            Map.of("reviewId", id.value()), "리뷰 작성 완료"));
    }

    @GetMapping("/products/{productId}/reviews")
    public ResponseEntity<CommonResponse<ReviewListResponse>> list(
        @PathVariable String productId,
        @RequestParam(defaultValue = "LATEST") ReviewSort sort,
        @RequestParam(required = false) Boolean hasImage,
        @RequestParam(required = false) @Min(1) @Max(5) Integer ratingMin,
        @RequestParam(defaultValue = "20") @Min(1) @Max(100) int limit,
        @RequestParam(defaultValue = "0") int offset
    ) {
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            query.search(new ProductId(productId), sort, hasImage, ratingMin, limit, offset),
            "조회 성공"));
    }

    @PostMapping("/reviews/{reviewId}/helpful")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<CommonResponse<Map<String, Boolean>>> helpful(
        @PathVariable String reviewId, Authentication auth
    ) {
        var active = toggleHelpful.handle(new ReviewId(reviewId), new UserId(auth.getName()));
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK,
            Map.of("active", active), active ? "도움돼요 표시" : "취소"));
    }

    @PostMapping("/reviews/{reviewId}/report")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<CommonResponse<Void>> reportReview(
        @PathVariable String reviewId,
        @Valid @RequestBody ReportRequest req, Authentication auth
    ) {
        report.handle(new ReviewId(reviewId), new UserId(auth.getName()),
            req.reason(), req.detail());
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK, "신고 접수"));
    }
}

public record CreateReviewRequest(
    @NotBlank String orderItemId,
    @Min(1) @Max(5) int rating,
    @NotBlank @Size(min = 1, max = 2000) String content,
    @Size(max = 5) List<@Pattern(regexp = "^https://.*") String> imageUrls
) {}

public record ReportRequest(
    @NotBlank String reason, @Size(max = 1000) String detail
) {}
```

---

## 6. Moderation Admin

```java
@RestController
@RequestMapping("/admin/reviews")
@PreAuthorize("hasAnyRole('ADMIN', 'MASTER')")
public class ReviewModerationController {

    @GetMapping("/reports/pending")
    public ResponseEntity<...> pendingReports(@RequestParam(defaultValue = "0") int page) { ... }

    @PostMapping("/{id}/hide")
    public ResponseEntity<...> hide(@PathVariable String id, @RequestBody HideReason reason) { ... }

    @PostMapping("/{id}/restore")
    public ResponseEntity<...> restore(@PathVariable String id) { ... }

    @PostMapping("/{id}/permanent-delete")
    public ResponseEntity<...> hardDelete(@PathVariable String id) { ... }
}
```

---

## 7. 함정 모음

### 함정 1 — 구매 확인 누락
누구나 리뷰 가능 = 스팸. `order_item_id` FK 로 구매 검증.

### 함정 2 — (user, product) UNIQUE 없음
같은 사용자가 같은 상품에 여러 리뷰. DB UNIQUE 필수.

### 함정 3 — 평점 매번 전체 재계산
인기 상품 = O(N). 증분 update 또는 batch.

### 함정 4 — `helpful_count` 직접 UPDATE
hot review 의 동시 클릭 = lock 대기. Redis INCR 또는 atomic UPDATE.

### 함정 5 — 신고 무제한
중복 신고 가능 = 어뷰즈. `(review_id, reporter_id) UNIQUE`.

### 함정 6 — 자동 숨김 threshold 너무 낮음
3명이 짜고 정상 리뷰 숨김 가능. 5~10 권장 + 수동 검토.

### 함정 7 — 이미지 외부 URL
사용자가 임의 URL 넣음 = XSS / phishing. **우리 CDN 도메인만 화이트리스트**.

### 함정 8 — content 의 HTML
저장은 raw, 출력 시 escape. 또는 markdown.

### 함정 9 — 리뷰 수정 무제한
판매 후 수정 = 평점 조작 가능. **30일 이내 + 평점 변경 N회 제한**.

### 함정 10 — `aggregate` 가 read-replica 미반영
read replica 에서 stats 읽는데 master 의 새 리뷰 반영 안 됨. **트랜잭션 master 에서 직접** 또는 sync 시간 감수.

---

## 8. 운영 체크리스트

- [ ] order_item FK + reviewable 검증
- [ ] (user, product) UNIQUE
- [ ] 평점 집계 — async + batch
- [ ] helpful_count Redis 카운터 (옵션)
- [ ] 자동 숨김 + 검토 큐
- [ ] 이미지 우리 CDN 만
- [ ] moderation admin 화면
- [ ] 신고 추세 모니터 (악의적 신고자 식별)

---

## 9. 관련

- [[product-crud]] — 상품 의 product_stats 비정규화
- [[order-stock]] — order_item 의 reviewable 상태
- [[file-upload-s3]] — 리뷰 이미지
- moderation (예정) — 자동 필터링
- [[api-design|↑ api-design hub]]
