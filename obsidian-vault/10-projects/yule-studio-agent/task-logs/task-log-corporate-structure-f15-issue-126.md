---
title: "F15 #126 회사 구조 + manifest 통일 + 플러그인 문서화"
kind: task-log
session_id: corporate-structure-f15
project: yule-studio-agent
created_at: 2026-05-11T00:00:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - ../decisions/decision-product-vs-marketing-cpo-cmo-separation-issue-126.md
  - ../knowledge/plugins/README.md
tags:
  - task-log
  - corporate-structure
  - manifest-unification
  - plugin-catalog
  - issue-126
---

# 목표

세 가지를 한 사이클 안에 마무리한다:

1. C-level (CTO / CPO / CMO / 등) 산하 부서 골격 + product/marketing 4-7
   역할 정착
2. F11 이전 `agent.json` 스키마와 F11 `manifest.json` 스키마 공존 종료
3. 11 개 플러그인의 도입 이유 / 동작 / 운영 가이드 카탈로그 (repo +
   vault 미러)

부수적으로 vault 명명 컨벤션 정리 (한 눈에 알아볼 수 있는 형식 박기).

# 진행 (commit 별)

| commit | 메시지 요약 | 결과 |
| --- | --- | --- |
| 1 | corporate org chart (C-level 매트릭스) | `policies/runtime/agents/corporate-org-chart.md` |
| 2 | product-agent CPO 산하 3 역할 | product-manager / user-researcher / growth-analyst |
| 3 | marketing-agent CMO 산하 4 역할 | growth-marketer / content-strategist / seo-specialist / brand-manager |
| 3.5 | 부서 경계 정책 + CPO/CMO 분리 결정 | `policies/.../department-boundary-policy.md` + vault decision 노트 |
| 4 | engineering-agent 7 role manifest.json 데이터 통합 | agent.json rich 데이터 흡수 (manifest.json 안으로) |
| 5 | loader 전환 + dept agent.json → manifest.json rename | 5 src 파일 + 2 dept 파일 git mv |
| 6 | role agent.json 일괄 삭제 + tests/policies/docs 참조 정리 | 7 파일 삭제 + 27 파일 참조 갱신, 회귀 47 passed |
| 7 | 11 플러그인 운영 카탈로그 | `policies/runtime/plugins/{README,*.md}` 12 파일 |
| 8 | vault 정리 (본 commit) | 7 노트 rename + plugin 미러 11 + 본 task-log + vault README |

# 결정 요약

- **product vs marketing 분리 유지** — 의도적. C-level 매핑 분리,
  Discord 채널 분리, A/B test 활성화 cross-review 등 7 항목 결정.
  본 vault 의 `decisions/decision-product-vs-marketing-cpo-cmo-separation-issue-126.md`
  참조.
- **manifest.json 단일 스키마** — 모든 부서 / 역할 / 플러그인이
  같은 파일명을 쓴다. 단 스키마는 위치 별로 다름:
  - `plugins/<id>/manifest.json` = F11 PluginManifest (엄격 검증)
  - `agents/<dept>/<role>/manifest.json` = F11 AgentManifest + 확장 필드
  - `agents/<dept>/manifest.json` = department config (members /
    participants / integrations / policies, untyped dict)
- **vault 명명 컨벤션** — `YYYY-MM-DD_<kind>_<topic-kebab>.md`.
  vault README 에 박음.

# 비결정 (다음 사이클로)

- hr-agent / finance-agent / sales-cs-agent / legal-agent 골격 (현재는
  org chart 에만 존재, manifest 미작성)
- PM skills catalog (pm-skills 참조 — 다음 commit 묶음)
- F15 governance 테스트 (manifest ↔ 정책 ↔ vault 일관성 검증)
- F4/F13 wiring follow-up (Discord post 자동화)
- CI merge guard (auto-merge gate 강화)

# Hard rails 확인

- 모든 plugin manifest 가 `paste_guard_required` 명시 (HIGH risk 제외)
- 7 개 role manifest 의 `module_path` 가 실제 import 경로 형식
- vault 노트는 PasteGuard 통과 후 작성 (본 task-log 도 동일)
- 모든 commit 메시지 한국어 3 섹션 (변경 이유 / 주요 변경 사항 / 비고)
- Co-Authored-By 라인 미부착 (사용자 메모리 규칙)

# 다음 행동

1. F15 잔여 commit (hr-agent + finance + sales-cs + legal 골격, PM
   skills catalog, governance 테스트, .env.example 갱신, memory feedback)
2. PR 생성 + Discord CI webhook 셋업 가이드 첨부
3. F4/F13 wiring 후속 commit
