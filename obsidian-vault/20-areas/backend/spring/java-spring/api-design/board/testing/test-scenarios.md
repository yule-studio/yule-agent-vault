---
title: "Test Scenarios — 8 영역 × 45 AC"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T14:55:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - testing
  - scenarios
---

# Test Scenarios — 8 영역 × 45 AC

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[testing|↑ testing hub]]**

> [[../requirements#2 완료 조건]] 의 45 AC × test 매핑.

---

## 1. 우선순위

🔴 必 / 🟡 권장 / 🟢 옵션 — [[../../signup/testing/test-scenarios#1]] 와 동일 기준.

---

## 2. Post CRUD

| AC | 시나리오 | 종류 | 우선 |
| --- | --- | --- | --- |
| AC-1 | 정상 작성 | 통합 | 🔴 |
| AC-2 | 작성자만 수정 (다른 user 403) | 통합 | 🔴 |
| AC-3 | soft delete (DELETED row 유지) | 통합 | 🔴 |
| AC-4 | DELETED 글의 댓글 보존 | 통합 | 🟡 |
| AC-5 | XSS sanitize (`<script>` reject) | 단위 | 🔴 |
| AC-6 | 비공개 board 404 | 통합 | 🔴 |
| AC-7 | 동시 작성 race | 통합 (멀티스레드) | 🟡 |

## 3. Comment

| AC | 시나리오 | 우선 |
| --- | --- | --- |
| AC-8 | flat 댓글 | 🔴 |
| AC-9 | 대댓글 (parent_id 매핑) | 🔴 |
| AC-10 | 대대댓글 자동 평탄화 | 🔴 |
| AC-11 | 부모 soft delete + content 마스킹 | 🔴 |
| AC-12 | tree 구성 (N+1 없이) | 🟡 |
| AC-13 | post.comment_count UPDATE | 🟡 |

## 4. Like / Bookmark

| AC | 시나리오 | 우선 |
| --- | --- | --- |
| AC-14 | 중복 좋아요 idempotent | 🔴 |
| AC-15 | 취소 시 row DELETE + counter | 🔴 |
| AC-16 | counter 1h batch sync | 🟡 |
| AC-17 | 북마크 별 패턴 | 🔴 |
| AC-18 | rate limit (60/min) | 🟡 |

## 5. Search / Sort / Pagination

| AC | 시나리오 | 우선 |
| --- | --- | --- |
| AC-19 | DB ILIKE 검색 | 🔴 |
| AC-20 | 3 정렬 (latest / hot / top) | 🔴 |
| AC-21 | cursor pagination 정확 | 🔴 |
| AC-22 | hot_score 시간 감쇠 검증 | 🟡 |
| AC-23 | block filter 적용 | 🔴 |
| AC-24 | HIDDEN / DELETED 제외 | 🔴 |

## 6. Taxonomy

| AC | 시나리오 | 우선 |
| --- | --- | --- |
| AC-25 | board 의 category 만 선택 | 🔴 |
| AC-26 | category 검증 (다른 board reject) | 🔴 |
| AC-27 | tag 자동 생성 (race 처리) | 🟡 |
| AC-28 | tag 필터 + board 필터 | 🟡 |
| AC-29 | 인기 태그 정렬 | 🟢 |

## 7. Attachment

| AC | 시나리오 | 우선 |
| --- | --- | --- |
| AC-30 | presigned URL 정상 | 🔴 |
| AC-31 | size / type 검증 | 🔴 |
| AC-32 | 미완료 업로드 cleanup (lifecycle) | 🟡 |
| AC-33 | post 삭제 시 S3 cleanup | 🟡 |
| AC-34 | presigned 5분 만료 reject | 🟡 |

## 8. Moderation

| AC | 시나리오 | 우선 |
| --- | --- | --- |
| AC-35 | 신고 INSERT | 🔴 |
| AC-36 | 중복 신고 silent | 🔴 |
| AC-37 | 5회 자동 HIDDEN | 🔴 |
| AC-38 | admin restore / delete | 🔴 |
| AC-39 | 차단 INSERT + cache invalidate | 🔴 |
| AC-40 | 차단된 user 글 list 에서 제외 | 🔴 |

## 9. Notification (옵션)

| AC | 시나리오 | 우선 |
| --- | --- | --- |
| AC-41 | 좋아요 알림 outbox INSERT | 🟡 |
| AC-42 | 댓글 알림 (post 작성자) | 🟡 |
| AC-43 | 대댓글 알림 (parent 작성자) | 🟡 |
| AC-44 | AFTER_COMMIT 만 발송 | 🟡 |
| AC-45 | preference OFF 시 skip | 🟡 |

---

## 10. Given-When-Then

```java
@Test
void post_create_xss_sanitized() {
    // Given
    var maliciousContent = "<script>alert('XSS')</script>안녕";

    // When
    var post = postService.create(new PostCreateCmd(boardId, ..., maliciousContent), authorId);

    // Then
    assertThat(post.content()).isEqualTo(maliciousContent);   // 저장은 raw
    var rendered = renderer.render(post.content());
    assertThat(rendered).doesNotContain("<script>");           // 렌더링은 sanitize
}
```

---

## 11. 관련

- [[testing|↑ hub]]
- [[../../signup/testing/test-scenarios|↗ signup scenarios]] — 패턴
- [[../requirements#2 완료 조건]] — AC source
