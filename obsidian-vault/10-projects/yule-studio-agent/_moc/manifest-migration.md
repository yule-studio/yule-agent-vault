---
title: "MOC — agent.json → manifest.json 통일 (F15)"
kind: knowledge
project: yule-studio-agent
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T15:30:00+09:00
tags:
  - moc
  - hub
  - manifest-migration
  - f15
related:
  - _moc.md
  - f15-corporate-structure.md
---

# MOC — agent.json → manifest.json 통일

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 hub |

## 이 hub 가 묶는 것

F15 사이클의 핵심 작업이었던 두 형식 (`agent.json` legacy + F11
`manifest.json`) 통일. 결정 / 통합 데이터 흐름 / 검증 결과를 한 곳에서
본다.

**[[_moc|↑ MOC 인덱스]]**

## 핵심 결정 / 정책

repo:

- `docs/manifest-migration.md` — 왜 / 어떻게 / 검증 / 뒤집기 조건 (단일
  진실 문서)
- `policies/runtime/plugins/README.md` — 11 플러그인 카탈로그
- `policies/runtime/vault/naming-convention.md` — vault 명명 컨벤션

## 관련 task-log

- [[../task-logs/task-log-corporate-structure-f15-issue-126]] — F15
  commit 4·5·6·9 가 통일 진행
- [[../task-logs/task-log-f15-smoke-test-non-discord]] — Discord 없는
  e2e smoke 로 통일 후 동작 검증

## 검증 결과

- 1772 pytest passed
- `yule doctor` → `OK agent context  agents/engineering-agent/manifest.json`
- engineer lifecycle (intake → approve → progress → complete) 정상

## 관련 hub

- [[f15-corporate-structure]] — 본 통일이 속한 사이클
- [[plugins-catalog]] — 같은 manifest 스키마를 쓰는 11 플러그인
