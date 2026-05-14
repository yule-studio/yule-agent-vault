---
title: "이슈 템플릿 — [TASK] 작업 / 설정"
kind: pattern
project: github-conventions
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:40:00+09:00
tags:
  - github
  - issue-template
  - task
  - setting
---

# 이슈 템플릿 — [TASK] 작업 / 설정

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 정리 |

**[[github-conventions|↑ GitHub 공통 작업 규칙]]**

---

## 1. 언제

기능 구현 (`✨ Feature`) 외의 **환경 설정 / 인프라 / 리팩토링 / 도구 도입**.
- 개발 환경 세팅
- linter / formatter / pre-commit hook
- 의존성 업그레이드 (보안 / 메이저)
- 폴더 구조 변경
- 코드 컨벤션 마이그레이션

---

## 2. 라벨 / 메타

- **labels**: `📃 Docs`, `⚙ Setting` (보통 동반)
- **assignees**: 자유 (빈 값도 OK)
- **title prefix**: `[TASK] `

---

## 3. 템플릿 (`.github/ISSUE_TEMPLATE/task.md`)

```markdown
---
name: "[TASK] Task (작업 / 설정)"
about: "기능 구현 외의 환경 설정, 인프라 구축, 리팩토링 등을 위한 템플릿입니다."
title: "[TASK] "
labels: ["📃 Docs", "⚙ Setting"]
assignees: []
---

## 📌 작업 개요
> 이번 작업의 목적을 한 줄로 요약해주세요.

## 📑 주요 작업 내용
- [ ]
- [ ]
- [ ]

## 🔗 관련 Jira 티켓 (있을 경우)
- 티켓 번호:

## 📝 비고
- 작업 시 주의사항이나 참고할 링크를 적어주세요.
```

---

## 4. 작성 가이드

### 4.1 작업 개요
**왜** 가 핵심. "그냥" / "있으면 좋아서" 는 X.
```
✅ "pre-commit hook 도입으로 lint 누락 PR 차단 (현재 주 2-3 회 발생)"
❌ "pre-commit hook 도입"
```

### 4.2 주요 작업 내용
실행 가능한 sub-task 로 분해. 각각 commit 가능한 단위.
```
- [ ] husky 설치 + package.json
- [ ] lint-staged 설정
- [ ] eslint / prettier 적용 범위
- [ ] CI 에서도 같은 lint 실행 (이중 검증)
- [ ] CONTRIBUTING.md 갱신
```

### 4.3 Jira / 외부 추적
- Jira / Linear / Notion 티켓 있으면 명시
- 양방향 link (GitHub issue ↔ external)

### 4.4 비고
- 차단 요인 (의존 작업)
- 작업 시간 예상
- 참고 PR / 문서

---

## 5. priority_boost

- `⚙ Setting` = **+25** — 개발 환경 세팅 (의외로 높음 — 모든 작업의 기반)
- `📃 Docs` = **-5**

합산 = **+20** 정도. 동반 라벨로 조정:
- `+ 🏗 infrastructure` (+30) — 인프라 변경
- `+ 🔨 refactor` (+0) — 리팩토링 단순
- `+ 🐞 BugFix` (+25) — bug 회피용 task

자세히 → [[pattern-label-rules]]

---

## 6. Feature vs Task — 어느 쪽?

| | Feature | Task |
| --- | --- | --- |
| 사용자가 보는 가치 | ✅ | ❌ |
| 코드 변경 | 도메인 | 인프라 / 설정 |
| 예 | "CSV export" | "ESLint 도입" |

→ 사용자가 보는 새 가치면 `✨ Feature`, 내부 / 인프라면 `[TASK]`.

---

## 7. 관련

- [[github-conventions]] — hub
- [[pattern-issue-feature]] — Feature 와 구분
- [[pattern-issue-cicd]] — CI/CD 변경
- [[pattern-label-rules]]
