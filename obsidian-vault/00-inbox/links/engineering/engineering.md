---
title: "Engineering 부서 — reference 인덱스"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T18:30:00+09:00
tags:
  - reference
  - links
  - engineering
  - cto
---

# Engineering (CTO 부서) — Reference 인덱스

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 7 역할 카탈로그 |

**[[../links|↑ links 카탈로그]]**

## 역할별 카탈로그

| 역할 | 진입점 | 핵심 영역 |
| --- | --- | --- |
| tech-lead | [[tech-lead]] | 시스템 설계 / 리뷰 / 합의 운영 / ADR |
| backend | [[backend]] | Java / Spring / JPA / API 설계 / 보안 |
| frontend | [[frontend]] | React / TypeScript / 상태 관리 / 접근성 |
| ai | [[ai]] | LLM / Claude / RAG / agent runtime / 평가 |
| devops | [[devops]] | Docker / K8s / CI/CD / 관측 / 시크릿 |
| qa | [[qa]] | 테스트 전략 / contract test / 회귀 |
| product-designer | [[product-designer]] | UI/UX / 디자인 시스템 / 사용자 흐름 |
| android | [[android]] | Kotlin / Jetpack Compose / 아키텍처 / KMP |

## 운영 규칙

본 부서의 reference 는 yule-studio-agent repo 의
`agents/engineering-agent/<role>/manifest.json` 의 `research_focus` 와
정합 유지. 새 역할 reference 추가 시 manifest 의 capabilities 와 호응.
