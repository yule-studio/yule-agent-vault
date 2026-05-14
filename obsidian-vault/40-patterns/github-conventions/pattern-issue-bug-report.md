---
title: "이슈 템플릿 — 🐞 Bug Report"
kind: pattern
project: github-conventions
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:35:00+09:00
tags:
  - github
  - issue-template
  - bug
---

# 이슈 템플릿 — 🐞 Bug Report

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 정리 |

**[[github-conventions|↑ GitHub 공통 작업 규칙]]**

---

## 1. 언제

기능 오류 / 예외 / 비정상 동작 / 회귀 발견 시.

---

## 2. 라벨 / 메타

- **labels**: `🐞 BugFix`
- **assignees**: `codwithyc` (기본)
- **title prefix**: `[BUG] `

---

## 3. 템플릿 (`.github/ISSUE_TEMPLATE/bug_report.md`)

```markdown
---
name: "🐞 Bug Report"
about: "버그를 발견했을 때 사용하세요."
title: "[BUG] "
labels: ["🐞 BugFix"]
assignees: [codwithyc]
---

## 🖥️ 환경 정보
- OS / Java 버전:
- 브라우저 / Postman 버전:

## 🔄 재현 절차
1.
2.
3.

## ✅ 기대 동작
>

## ❌ 실제 동작
>

## 📸 스크린샷 / 로그
>
```

---

## 4. 작성 가이드

### 4.1 환경 정보
정확한 버전 명시. "최신" / "어제" 금지.
```
✅ OS: macOS 14.5, Java: OpenJDK 21.0.2
❌ OS: 최신 macOS
```

### 4.2 재현 절차
**최소 재현 단계** (Minimal Reproducible). 5 단계 이하 권장.
- 외부 의존 (DB / API) = mock 으로 격리
- 입력 / state 명시

### 4.3 기대 vs 실제
- `기대` = 의도된 동작 (스펙 문서 / 합의)
- `실제` = 관찰된 동작 (스택트레이스 / 응답 / UI 상태)

### 4.4 스크린샷 / 로그
- 가능하면 영상 (gif) > 스크린샷 > 텍스트
- 로그는 ``` ``` 코드 블록
- 민감 정보 (토큰 / PII) 마스킹

---

## 5. priority_boost

`🐞 BugFix` = **+25** — 작업 우선순위 ↑ (자세히 → [[pattern-label-rules]]).

---

## 6. 후속 작업

1. 재현 검증 → 라벨 추가 (`bug-confirmed` 등)
2. 원인 분석 → 영향 범위
3. PR 작성 시 `closed #<issue-num>` 명시
4. 회귀 테스트 추가

---

## 7. 관련

- [[github-conventions]] — hub
- [[pattern-label-rules]] — 🐞 BugFix priority_boost
- [[pattern-workflow]] — 이슈 → PR 흐름
