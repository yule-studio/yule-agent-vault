---
title: "00-inbox — 분류 전 임시"
kind: knowledge
project: agent-inbox
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T18:30:00+09:00
tags:
  - inbox
  - index
---

# 00-inbox — 분류 전 임시

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.2.0.0 | 2026-05-12 | engineering-agent/tech-lead | 3 하위 영역 실제 노트로 채움 (links 17 / quick-notes 10 / unsorted 10) |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 — 빈 골격 |

**[[../index|↑ obsidian-vault 인덱스]]**

## 하위 영역

| 영역 | 진입점 | 현재 노트 |
| --- | --- | --- |
| links | [[links/links]] | 4 부서 (engineering 7 / product 3 / marketing 4) + general 4 → 17 reference 카탈로그 + 부서 인덱스 4 |
| quick-notes | [[quick-notes/quick-notes]] | 10 아이디어 (PR 자동 라벨 / 그래프 클러스터링 / cost tracker / 등) |
| unsorted | [[unsorted/unsorted]] | 10 노트 (관찰 3 / 후속 5 / 메타 2) |

## 운영 규칙

1. **임시 저장소**. 일주일 이상 inbox 에 남은 노트는 위험 — 분류 미루지
   말 것.
2. **분류 후 이주**:
   - 외부 자료 요약 → `30-resources/<category>/`
   - 개념 정리 → `20-areas/<area>/`
   - 코드 조각 → `50-snippets/<lang>/`
   - 프로젝트 관련 → `10-projects/<project>/`
   - 더 안 볼 노트 → `90-archive/`
3. **메타데이터 약식 OK**. inbox 단계에서는 frontmatter / 컨벤션 엄격
   적용 안 함. 분류 시점에 정식 컨벤션으로 변환.
4. `links/` 는 예외 — 영구 reference 카탈로그라 inbox 에 있어도 갱신
   계속.

## 자동화 에이전트 진입

에이전트가 `kind` 를 결정 못 할 때 `00-inbox/unsorted/` 로 라우팅
(`export.py:_yule_vault_kind_to_folder` 의 fallback). 운영자가 inbox
스윕 시점에 적절한 폴더로 이주.

## 관련

- [[../index|↑ obsidian-vault 인덱스]]
- [[../../CLAUDE|운영 규칙 (CLAUDE.md)]]
