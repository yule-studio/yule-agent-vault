---
title: "Runbook — board 사고 대응"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T15:15:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - operations
  - runbook
---

# Runbook — board 사고 대응

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[operations|↑ operations hub]]**

> board 특화 사고 시나리오. signup runbook 베이스 + board 특이성.

---

## 1. 시나리오별 대응

### 1.1 Counter mismatch (Redis ↔ DB)

```
증상: post.like_count 가 actual post_likes count 와 다름

대응:
1. Redis vs DB count 차이 측정
2. 차이 < 5% — batch sync 기다림 (1h)
3. 차이 > 5% — 강제 sync 실행
   SQL: UPDATE posts SET like_count = (SELECT count(*) FROM post_likes WHERE post_id = posts.id)
4. Redis 값도 갱신
5. 원인 분석 (AFTER_COMMIT listener fail / batch job 지연)
```

### 1.2 XSS 발견 (사용자 신고)

```
1. 즉시 — 해당 글 status=HIDDEN
2. 패턴 분석 — 다른 글에도 같은 XSS 시도?
3. SanitizePolicy 갱신 (whitelist 강화)
4. 영향받은 글 일괄 sanitize (재렌더링)
5. PIPC 신고 검토 (사용자 PII 영향 시)
```

### 1.3 S3 storage 폭증

```
증상: S3 monthly bill 50% ↑

분석:
1. S3 inventory analyze — 어떤 prefix 가 큰가
2. user 별 storage SUM(size_bytes)
3. orphan 파일 (post 연결 X) 비율

대응:
1. 큰 user 추적 (abuse?)
2. lifecycle 정책 강화 (옛 파일 archive / 삭제)
3. 사용자 별 storage 한도 (per-user 10GB 등)
```

### 1.4 Spam report flood

```
증상: 분당 100+ 신고 발생

분석:
1. 신고자 IP 분포 (조직적?)
2. target 분포 (특정 user 만?)
3. reason 분포

대응:
1. WAF 또는 rate limit 강화
2. 같은 IP 의 다중 신고 = 한 표로 가중치 처리
3. admin manual review queue 분리 (suspicious 우선)
```

### 1.5 Notification outbox backlog

```
증상: notification_outbox.pending > 10000

분석:
1. Worker liveness
2. FCM / APNs 응답 시간
3. DB lock 의심

대응:
1. Worker 재시작
2. Worker 인스턴스 ↑ (scale-out)
3. FCM 일시 장애 시 — retry backoff 강화
```

### 1.6 인기 글 폭발 (DB hot row)

```
증상: 특정 post 의 view / like 폭증 → DB lock

대응:
1. 해당 post 의 counter 가 Redis 만 (DB UPDATE skip)
2. Read replica 로 SELECT 분산
3. cache 강화 (post 본문 5min TTL)
4. CDN — 정적 응답 cache (옵션)
```

---

## 2. Smoke Test (배포 후)

```bash
# 1. 게시판 list
curl https://api.example.com/api/v1/boards/free/posts

# 2. 글 생성
curl -X POST .../posts -H "Auth: ..." -d "{...}"

# 3. 댓글
curl -X POST .../posts/.../comments

# 4. 좋아요
curl -X POST .../posts/.../like

# 5. 검색
curl ".../posts/search?q=test"
```

---

## 3. 일반 대응

자세히: [[../../signup/operations/runbook|↗ signup runbook]] — 5xx / OOM / DB pool 등 일반 시나리오.

---

## 4. 관련

- [[operations|↑ hub]]
- [[../../signup/operations/runbook|↗ signup runbook]]
- [[observability]]
