---
title: "GitHub 공통 작업 규칙 (Hub)"
kind: pattern
project: github-conventions
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:30:00+09:00
tags:
  - github
  - convention
  - issue-template
  - pr-template
  - label
  - hub
---

# GitHub 공통 작업 규칙 — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 이슈 6 종 + PR + 라벨 + 워크플로우 분리 정리 |

**[[../patterns|↑ 40-patterns]]** · **[[../../index|↑↑ vault]]**

> 이 노트는 사용자 (`codwithyc`) 의 **모든 GitHub 프로젝트 공통** 협업 규칙의 hub.
> RAG / CAG 로 각 에이전트가 참조하므로 **하나의 source of truth**.

---

## 1. 사용 의도

- 이슈 / PR 작성 시 일관된 구조
- 라벨 기반 자동 우선순위 (priority_boost)
- 신규 기여자 / 자동화 에이전트의 onboarding
- `.github/ISSUE_TEMPLATE/`, `.github/PULL_REQUEST_TEMPLATE/`, `.github/labels.yml` 의 원본

---

## 2. 이슈 템플릿 6 종

| 종류 | 노트 | 라벨 |
| --- | --- | --- |
| Bug Report | [[pattern-issue-bug-report]] | 🐞 BugFix |
| Feature Request | [[pattern-issue-feature]] | ✨ Feature |
| CI/CD Config | [[pattern-issue-cicd]] | 🌏 Deploy |
| Docs Request | [[pattern-issue-docs]] | 📃 Docs |
| Release / Deployment | [[pattern-issue-release]] | 🌏 Deploy |
| Task (작업 / 설정) | [[pattern-issue-task]] | 📃 Docs, ⚙ Setting |

각 노트 = `.github/ISSUE_TEMPLATE/*.md` 의 그대로 + 작성 가이드.

---

## 3. PR 템플릿

- [[pattern-pr-template]] — `.github/PULL_REQUEST_TEMPLATE.md` 표준

---

## 4. 라벨 + Priority Boost

- [[pattern-label-rules]] — 17 개 라벨 + `priority_boost` JSON

라벨 = 자동 우선순위 계산의 입력. 에이전트가 이슈 / PR 처리 순서를 결정할 때 참조.

---

## 5. 워크플로우 (이슈 → PR → merge)

- [[pattern-workflow]] — 이슈 생성 → 작업 → PR → review → merge 의 표준 흐름

---

## 6. 적용 위치

| 위치 | 파일 |
| --- | --- |
| GitHub repo | `.github/ISSUE_TEMPLATE/bug_report.md` 등 |
| GitHub repo | `.github/PULL_REQUEST_TEMPLATE.md` |
| GitHub repo | `.github/labels.yml` (선택 — actions-labeler 등) |
| 에이전트 | 이 vault 의 노트 (RAG / CAG) |

---

## 7. assignee 기본값

대부분 이슈의 기본 assignee = `codwithyc`.
(자동화 에이전트가 작업 분배 시 override 가능)

---

## 8. 제목 prefix 규칙

| 종류 | prefix |
| --- | --- |
| Bug | `[BUG] ` |
| Feature | (자유) |
| CI/CD | `[CI/CD] ` |
| Docs | `[DOCS] ` |
| Release | `[RELEASE] v` |
| Task | `[TASK] ` |

---

## 9. 관련

- [[../../20-areas/devops/github-actions/github-actions]] — GitHub Actions / CI 워크플로우
- [[../../20-areas/devops/devops]] — devops area
- [[../patterns|↑ 40-patterns]]
- [[../../index|↑↑ vault]]
