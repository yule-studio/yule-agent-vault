---
title: "auth §12 — 흔한 함정 (Hub)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:40:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - pitfalls
  - hub
---

# auth §12 — 흔한 함정 (Hub)

**[[../signup|↑ signup hub]]**  ·  ← [[../implementation-order]]

> auth 도메인 구현 시 흔한 실수 / 안티패턴. **코드 리뷰 / 회귀 방지** 체크리스트.
> 각 함정은 **무엇 / 왜 위험 / 해결** 4구조.

---

## 1. 카테고리

| 노트 | 무엇 | 함정 수 |
| --- | --- | --- |
| [[database-pitfalls]] | DB schema / 인덱스 / 정합성 | 7+ |
| [[security-pitfalls]] | 비밀번호 / 평문 / 토큰 | 10+ |
| [[transaction-pitfalls]] | 트랜잭션 / 동시성 / 외부 호출 | 6+ |
| [[domain-pitfalls]] | 도메인 모델 / 검증 / VO | 8+ |
| [[operations-pitfalls]] | UX / Rate limit / 정책 | 8+ |

---

## 2. 왜 함정 카탈로그 필수

**왜 모은 카탈로그 필요**
- 새 합류자 학습 곡선 ↓ — "이거 하면 안 됨" 명시.
- 코드 리뷰 체크리스트 — PR 마다 확인.
- 회귀 발견 — 옛 사고가 다른 모양으로 재발.

**안 하면 무슨 문제**
- 같은 함정에 반복 빠짐.
- 사고 후 postmortem 만 → 다음 사고는 다른 곳.

**대안과 왜 안 됨**
- 코드 안 주석 — scatter, 검색 어려움.
- 사고 wiki — 너무 detail, 정작 PR 단계 활용 X.

**트레이드오프**
- maintenance — 카탈로그 outdated 위험.
- 정기 update + 신규 함정 추가.

---

## 3. 코드 리뷰 체크리스트 (요약)

PR 마다 확인:

### 3.1 DB

- [ ] 새 UNIQUE / FK / CHECK 인덱스의 race condition 검토
- [ ] `lower(email)` UNIQUE 가 잘못 유지되는지
- [ ] schema 변경 시 expand/contract 적용
- [ ] Flyway V 번호 충돌 없는지

### 3.2 보안

- [ ] DTO 가 password 필드 가지면 toString 마스킹
- [ ] 응답 DTO 에 password / password_hash / JWT 노출 X
- [ ] Hibernate SQL log level WARN+
- [ ] `@JsonIgnore` 누락된 sensitive 필드 없는지
- [ ] secret hardcoded X

### 3.3 트랜잭션

- [ ] `@Transactional` 안에 외부 API 호출 X
- [ ] `@TransactionalEventListener` phase 검토 (AFTER_COMMIT 권장)
- [ ] self-invocation 없는지
- [ ] AFTER_COMMIT 의 dual-write 없는지

### 3.4 도메인

- [ ] 도메인이 `@Entity` 와 결합 X
- [ ] Setter 노출 X (의미 있는 메서드만)
- [ ] Aggregate 경계 검토
- [ ] VO 의 compact constructor 검증

### 3.5 검증

- [ ] 모든 `@RequestBody` 에 `@Valid`
- [ ] Bean Validation message 명시
- [ ] 도메인 VO 의 이중 검증

### 3.6 운영

- [ ] Rate limit 적용
- [ ] 새 endpoint 의 SecurityConfig 추가
- [ ] 메트릭 / audit log 추가
- [ ] 시나리오 표 갱신

---

## 4. 발견 → 학습 흐름

```
1. PR 단계 발견 (best)
   → checklist 활용 + 즉시 수정

2. 코드 머지 후 발견
   → hotfix + 함정 카탈로그 추가

3. 운영 사고 발견 (worst)
   → postmortem + 카탈로그 + drill
```

---

## 5. 관련

- [[../signup|↑ signup hub]]
- [[../implementation-order]] — 이전 (§11)
- [[../../pitfalls]] — Java Spring 전반 함정
- [[../testing/test-scenarios]] — 시나리오 검증
