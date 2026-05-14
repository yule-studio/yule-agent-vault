---
title: "이슈 템플릿 — 📑 [Docs] 문서 요청/개선"
kind: pattern
project: github-conventions
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:38:00+09:00
tags:
  - github
  - issue-template
  - docs
---

# 이슈 템플릿 — 📑 [Docs] 문서 요청 / 개선

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 정리 |

**[[github-conventions|↑ GitHub 공통 작업 규칙]]**

---

## 1. 언제

프로젝트 문서 (가이드 / 정책 / 전략 / API spec) 신규 작성 또는 개선.
- README 부족 / 옛 정보
- 신규 기여자 onboarding 자료
- 결정 기록 / 아키텍처 문서

---

## 2. 라벨 / 메타

- **labels**: `📃 Docs`, `help wanted` (옵션)
- **assignees**: `codwithyc`
- **title prefix**: `[DOCS] `

---

## 3. 템플릿 (`.github/ISSUE_TEMPLATE/docs_request.md`)

```markdown
---
name: "📑 [Docs] 문서 요청/개선"
about: "프로젝트 내 문서(가이드, 정책, 전략 등)를 새로 만들거나 개선하고 싶을 때 사용하세요."
title: "[DOCS] "
labels: ["📃 Docs", "help wanted"]
assignees: [codwithyc]
---

## 📌 문서 유형
- [ ] 신규 작성
- [ ] 기존 문서 개선

## 📝 문서 제목
> 작성하거나 개선할 문서 파일명 (예: `BRANCH_STRATEGY.md`)

```
예) 브랜치 전략
```

## 🚀 배경 및 목적
> 왜 이 문서가 필요한가요? 어떤 문제를 해결하나요?

```
- README만으로는 세부 브랜치 전략을 찾기 어려움
- 신규 기여자 온보딩 시 참고 자료가 필요함
```

## 📑 제안 내용 (목차/주요 항목)
> 문서에 담을 주요 섹션이나 상세 내용을 적어주세요.

```
1. 브랜치 종류 및 역할
2. PR 머지·릴리즈 워크플로우
3. 네이밍 컨벤션
4. 예시 다이어그램
```

## 🔗 참고 자료
> 기존 문서나 외부 레퍼런스 URL

- https://example.com/branch-strategy
- docs/OLD_BRANCH.md

## 🔖 우선순위
- [ ] High
- [ ] Medium
- [ ] Low
```

---

## 4. 작성 가이드

### 4.1 신규 vs 개선
- **신규** — 비슷한 문서 검색 후 (`grep` / Obsidian search) 중복 없음 확인
- **개선** — 기존 파일 path 명시 + 변경 의도

### 4.2 배경 및 목적
"왜" 가 명확해야 함. 단순 "필요해서" 는 X.
```
✅ "신규 기여자가 첫 PR 까지 1시간 안에 도달하지 못함 (현재 평균 3시간) → onboarding 문서 필요"
❌ "문서가 필요함"
```

### 4.3 제안 목차
- 30-60 분 안에 읽을 수 있는 분량 (5-10 섹션)
- 한 문서 = 한 주제 (긴 문서는 분할)
- vault 의 [[../../index|폴더 컨벤션]] 따름

### 4.4 우선순위
- **High** — 차단 요인 (배포 / onboarding 불가)
- **Medium** — 개선 (시간 절약)
- **Low** — nice-to-have

---

## 5. priority_boost

`📃 Docs` = **-5** — 일반 작업보다 낮은 우선순위. 문서는 backlog 후순위.

⚠️ 단, **차단 (블로커) 문서** 는 추가 라벨 (`🐞 BugFix` 또는 `🏗 infrastructure`) 으로 boost.

---

## 6. 후속 작업

1. 문서 위치 결정 — repo 의 `docs/` 또는 [[../../index|vault 폴더]] (`20-areas/` / `30-resources/`)
2. PR 에 변경 사항 미리보기 (스크린샷 / preview link)
3. README 의 link 갱신
4. 관련 vault 노트와 cross-link

---

## 7. 관련

- [[github-conventions]] — hub
- [[../../index|Obsidian 폴더 컨벤션]]
- [[pattern-label-rules]] — 📃 Docs 의 -5
