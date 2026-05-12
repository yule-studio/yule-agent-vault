---
title: "Notion import (19 페이지) + 00-inbox 콘텐츠 채우기"
kind: task-log
session_id: notion-import-inbox-fill-2026-05-12
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: completed
created_at: 2026-05-12T19:00:00+09:00
issue_number: 126
related:
  - task-log-vault-readme-rename-and-tree-connect.md
  - ../../../notion-import/notion-import.md
tags:
  - task-log
  - notion-import
  - vault
  - inbox
  - links
  - issue-126
---

# Notion import + 00-inbox 콘텐츠 채우기

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

## 목표

사용자 요청 3 건:

1. 공개 Notion 페이지 (`yuchan-log.notion.site/5675c9d0...`) 의 내용을
   vault 로 일괄 import.
2. `00-inbox/` 의 3 하위 영역 (links / quick-notes / unsorted) 에 각각
   최소 10 개 상세 콘텐츠.
3. `links/` 는 에이전트별 reference URL 카탈로그 (GitHub / 블로그 / docs).

## 진행

### 1. Notion import (19 페이지)

- `loadPageChunk` 공개 API 로 19 sub-page fetch (`/api/v3/loadPageChunk`).
  sandbox outbound 차단 우려했지만 notion.site 는 통과.
- Notion block 트리 → Markdown 변환기 작성 (page / header /
  bulleted_list / quote / callout / code / column_list / collection_view
  등 처리).
- 결과: `10-projects/notion-import/` 에 19 파일 + `notion-import.md` 인덱스.
  파일 예: `전체-포스팅.md`, `어째서-돼지인가.md`, `main-project.md`.
- 인덱스에 추천 이주 위치 표 포함 (book → 30-resources/books, 자격증 →
  20-areas/career/certificates, 프로젝트 → 10-projects/<name>).

알려진 한계:
- `collection_view` (Notion DB) 는 별도 API 필요 — placeholder 로만 표시
- 200 blocks 이상 긴 페이지는 첫 청크만 import — 누락 시 follow-up
- 이미지 / 첨부는 Notion S3 URL 그대로 (TTL 위험)

### 2. links/ 부서 hierarchy 도입

처음에 flat (`tech-lead-references.md` / `backend-engineer-references.md`)
로 작성했지만 사용자 피드백 ("links 안에 모든 링크 추가는 비효율 —
폴더링이 좋겠다") 에 따라 즉시 재구성:

```
links/
├── links.md (top index)
├── engineering/  (CTO 산하 7 역할)
│   ├── engineering.md (부서 인덱스)
│   ├── tech-lead.md / backend.md / frontend.md / ai.md /
│   ├── devops.md / qa.md / product-designer.md
├── product/  (CPO 산하 3 역할)
│   ├── product.md
│   ├── product-manager.md / user-researcher.md / growth-analyst.md
├── marketing/  (CMO 산하 4 역할)
│   ├── marketing.md
│   ├── growth-marketer.md / content-strategist.md /
│   ├── seo-specialist.md / brand-manager.md
└── general/  (전부서 공통)
    ├── general.md
    ├── system-design.md / books.md / communities.md / ai-tools.md
```

각 카탈로그에 10-15 reference 링크 (공식 docs / GitHub / 권위 블로그).
총 **17 카탈로그 + 5 인덱스 = 22 파일**.

### 3. quick-notes/ 10 아이디어

운영 중 떠올린 후속 작업 후보 10 개:

- idea-pr-auto-label-by-scope (PR 자동 라벨)
- idea-vault-graph-clustering (그래프 hub 색)
- idea-cost-tracker-per-cycle (사이클 토큰/cost 추적)
- idea-discord-status-board (#봇-상태 rich embed)
- idea-mistake-ledger-dashboard
- idea-vault-search-cli
- idea-skill-codegen-from-prompts
- idea-cron-mvp-tasks
- idea-notion-incremental-sync
- idea-agent-self-improvement-loop

각 노트 = 아이디어 / 동기 / 구현 후보 / 성공 신호 / 블로커 5 섹션.

### 4. unsorted/ 10 노트

관찰 3 / 후속 5 / 메타 2 = 10:

- observation-bot-noise-pattern / observation-vault-graph-isolated-nodes
  / observation-anthropic-sandbox-discord-block
- followup-f15-governance-test / followup-hr-finance-skeletons /
  followup-discord-webhook-secret-setup / followup-pm-skills-catalog /
  followup-engineering-knowledge-collector
- meta-commit-splitting-effect / meta-100-percent-operational-framework

각 노트 = 관찰/후속 / 근거 / 정책 영향 / 후속 4 섹션.

### 5. 인덱스 갱신

- `00-inbox/inbox.md` v.2.0.0 — 3 하위 영역의 실제 콘텐츠 수 반영
- `links/links.md` v.2.0.0 — 부서 hierarchy 도입
- 4 부서 인덱스 (engineering.md / product.md / marketing.md / general.md)
- `quick-notes/quick-notes.md` / `unsorted/unsorted.md` 신설

## 결정 요약

- **부서 hierarchy 도입** — flat 구조가 노트 수 증가 시 검색 불가. CTO/CPO/
  CMO + general 4 그룹으로 묶음. yule-studio-agent 의 부서 매트릭스와 1:1.
- **각 폴더의 인덱스 파일 = 폴더명.md** — README 가 아니라 폴더명으로 가서
  Obsidian 그래프 라벨 식별 가능.
- **inbox 안의 quick-notes / unsorted 는 실제 콘텐츠 채움** — 임시 영역
  이지만 사용자 요청에 따라 10 개씩 의미 있는 내용으로. 운영 중 떠오른
  아이디어 / 관찰 / 후속을 누적.

## 비결정 (의도적)

- `collection_view` (Notion DB) 깊은 import 는 후속. 필요 시 별도 API
  호출 추가.
- Notion 이미지 / 첨부는 vault 안 로컬 다운로드 안 함 — Notion S3 URL
  유지 (TTL 끊기면 broken image). 중요 자료는 사용자가 직접 다운로드 권장.
- `links/` 의 카탈로그가 추후 너무 커지면 `30-resources/` 로 마이그레이션
  검토.

## 다음 행동

1. vault git commit + push (`yule-studio/yule-agent-vault`)
2. Notion import 노트들을 사용자가 검토하고 적절한 위치로 분류
3. quick-notes 의 채택된 아이디어는 `_moc/` hub 로 이주

## 통계

- Notion import: 19 페이지 + 1 인덱스 = **20 파일**
- inbox/links: 17 카탈로그 + 5 인덱스 = **22 파일**
- inbox/quick-notes: 10 아이디어 + 1 인덱스 = **11 파일**
- inbox/unsorted: 10 노트 + 1 인덱스 = **11 파일**
- 인덱스 갱신: inbox.md (v.2.0.0)
- **총 65 파일 신설 / 갱신**
