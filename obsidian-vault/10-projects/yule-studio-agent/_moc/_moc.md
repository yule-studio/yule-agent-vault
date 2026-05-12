# _moc/ — Map of Content (주제 hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 MOC hub README |

## 이 폴더가 묶는 것

폴더 (decisions / research / task-logs / knowledge) 는 **kind 기준
정리**. 본 `_moc/` 는 **주제 기준 정리** — 한 사이클 / 한 이슈에 묶이는
여러 kind 의 노트를 한 곳에서 본다.

**[[../index|↑ 상위 README]]**

## 현재 hub (위가 최신 사이클)

### F15 (2026-05-11 ~)

- [[f15-corporate-structure]] — CTO / CPO / CMO 부서 골격 + 의도적 분리
- [[manifest-migration]] — agent.json → manifest.json 통일 (4 commit)
- [[plugins-catalog]] — F11 11 플러그인 운영 카탈로그

### 운영 인프라

- [[issue-73-tech-lead-runtime]] — tech-lead runtime loop 4 단계 (#73)
- [[discord-ci-strategy]] — Discord ↔ CI 알림 경계 + CD 책임 분리
- [[engineering-knowledge-surface]] — share_scope + retrieval 강화 (#69)

### 외부 통합

- [[hermes-yule-integration]] — Hermes agent → Yule 흡수 결정 (#59)

## 운영 규칙

1. 새 사이클 / 큰 이슈 단위로 새 hub 노트 1 개 생성.
2. 그 사이클의 모든 decision / research / task-log 를 본 hub 에
   [[link]]. 폴더 README 와 cross-link.
3. hub 자체에는 결정을 내리지 않는다 — 결정은 `decisions/`, 진행은
   `task-logs/`, 자료는 `research/`. hub 는 **인덱스 + 가지** 만.
4. 사이클 종료 시 hub 의 frontmatter `status: completed` 로 갱신.
   superseded 되면 후속 hub 링크를 본문 첫 줄에.

## 폴더 README 가지

- [[../decisions/decisions|decisions/]]
- [[../research/research|research/]]
- [[../task-logs/task-logs|task-logs/]]
- [[../knowledge/knowledge|knowledge/]]
