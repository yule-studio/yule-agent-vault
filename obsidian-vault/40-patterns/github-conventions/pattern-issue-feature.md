---
title: "이슈 템플릿 — ✨ Feature Request"
kind: pattern
project: github-conventions
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:36:00+09:00
tags:
  - github
  - issue-template
  - feature
---

# 이슈 템플릿 — ✨ Feature Request

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 정리 |

**[[github-conventions|↑ GitHub 공통 작업 규칙]]**

---

## 1. 언제

새 기능 / 개선 요청. UX / API / 도메인 모델의 신규 추가.

---

## 2. 라벨 / 메타

- **labels**: `✨ Feature`
- **assignees**: `codwithyc`
- **title prefix**: 자유 (의도가 명확하면 OK)

---

## 3. 템플릿 (`.github/ISSUE_TEMPLATE/feature_request.md`)

```markdown
---
name: "✨ [Feature] request"
about: "새 기능 요청 / 개선 제안"
title: ""
labels: ["✨ Feature"]
assignees: [codwithyc]
---

## 어떤 기능인가요?
> 추가하려는 기능에 대해 간결하게 설명해주세요

## 작업 상세 내용
- [ ]

## 참고할만한 자료(선택)
```

---

## 4. 작성 가이드

### 4.1 어떤 기능인가요?
**한 문장 정의** + **사용자 시나리오** 1 개.
```
✅ "관리자가 사용자 목록을 CSV 로 export 할 수 있다."
   - 시나리오: 백오피스에서 통계 분석을 위해 사용자 목록 다운로드
❌ "export 기능"
```

### 4.2 작업 상세 내용
체크박스로 sub-task 분해. PR 1 개 단위로 자를 수 있게.
```
- [ ] API: GET /admin/users/export (CSV)
- [ ] 권한: 관리자 role 검증
- [ ] UI: 다운로드 버튼
- [ ] 테스트: 빈 목록 / 10K 행 / 권한 없음
```

### 4.3 참고할만한 자료
- 디자인 시안 (Figma URL)
- 기존 유사 기능 PR
- 외부 라이브러리 / 명세
- 비슷한 서비스 (스크린샷)

---

## 5. priority_boost

`✨ Feature` = **+10** (자세히 → [[pattern-label-rules]]).

다른 라벨 동반 시 합산:
- `+ 🎯 core` (+25) → 코어 기능
- `+ 📬 api` (+15) → API 추가
- `+ 🗄 schema` (+25) → DB 스키마 변경

---

## 6. 후속 작업

1. discuss → 디자인 결정 → [[../../../10-projects/<project>/decisions/]]
2. 작업 분해 → sub-issue (필요 시)
3. PR 작성 — `closed #<issue-num>`
4. 회귀 테스트 + 문서 (`📃 Docs` 이슈 별도)

---

## 7. 관련

- [[github-conventions]] — hub
- [[pattern-label-rules]] — ✨ Feature priority
- [[pattern-issue-task]] — 기능 외 작업
- [[pattern-workflow]]
