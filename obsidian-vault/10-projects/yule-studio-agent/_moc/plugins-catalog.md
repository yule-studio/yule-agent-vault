---
title: "MOC — F11 플러그인 운영 카탈로그 (11 개)"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T15:30:00+09:00
tags:
  - moc
  - hub
  - plugins
  - f11
related:
  - _moc.md
  - manifest-migration.md
---

# MOC — F11 플러그인 운영 카탈로그

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 hub |

## 이 hub 가 묶는 것

F11 (#102 / #150) 에서 도입된 11 개 플러그인의 도입 이유 / 동작 / 운영
가이드. repo `policies/runtime/plugins/` 와 vault `knowledge/plugins/`
양쪽에 미러 — repo 가 권위.

**[[_moc|↑ MOC 인덱스]]**

## 11 플러그인 (kind 별)

### guard

- [[../knowledge/plugins/paste-guard]] — outbound secret/PII 마스킹
- [[../knowledge/plugins/lsp-preflight]] — 정적 분석 사전 차단
- [[../knowledge/plugins/tool-call-gate]] — 5×5 autonomy × risk 매트릭스

### learning

- [[../knowledge/plugins/hookify]] — preflight/completion/postmortem +
  mistake ledger
- [[../knowledge/plugins/claude-mem]] — 5-소스 long-term memory unifier

### exploration

- [[../knowledge/plugins/repo-map]] — 작업 시작 전 레포 지도 캐시

### delivery

- [[../knowledge/plugins/discussion-response]] — Discord 응답 표준 양식
- [[../knowledge/plugins/live-llm-editor]] — F4 live coding executor
- [[../knowledge/plugins/live-research-provider]] — F5 live research
- [[../knowledge/plugins/auto-merge-decider]] — PR 자동 머지 결정
- [[../knowledge/plugins/obsidian-vault-push]] — vault 자동 push

## 운영 흐름

1. 운영자가 플러그인을 켜고 끌 때 → vault 의 해당 노트 + repo
   `policies/runtime/plugins/<id>.md` 양쪽 확인
2. 새 플러그인 추가 → `plugins/<id>/manifest.json` (F11 schema) +
   repo `policies/runtime/plugins/<id>.md` + vault
   `knowledge/plugins/<id>.md` 3 개 동시 land
3. 회귀: `tests/extension/test_plugin_manifest_*`

## 관련 hub

- [[manifest-migration]] — 같은 PluginManifest 스키마
- [[f15-corporate-structure]] — F15 commit 7 에서 본 카탈로그 작성
