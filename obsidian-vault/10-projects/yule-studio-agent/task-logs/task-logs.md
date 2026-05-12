# task-logs/ — 작업 로그

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 task-logs README hub |

## 이 폴더가 묶는 것

issue / 사이클 단위 **작업 진행 기록**. 무엇을 했고 / 왜 그렇게 했고 /
무엇이 막혔고 / 다음 행동은 무엇인지의 시계열. 결정 / 리서치 / 코드
변경의 연결점.

**[[../index|↑ 상위 README]]**

## 파일명 컨벤션

```
task-log-<topic-kebab>[-issue-<n>].md
```

## 현재 노트 (위가 최신)

- [[task-log-f15-smoke-test-non-discord]] — Discord 없는 e2e smoke
  (session 7c922f0a8e6d, 2026-05-12)
- [[task-log-corporate-structure-f15-issue-126]] — F15 corporate
  structure 사이클 진행 (commit 1~10)
- [[task-log-company-runtime-knowledge-surface]] — knowledge surface
  운영 통합 작업
- [[task-log-discord-ci-design]] — Discord CI 디자인 작업
- [[task-log-tech-lead-runtime-loop-issue-73]] — tech-lead runtime
  loop 4 단계 (#73)
- [[task-log-hermes-tech-lead-issue-59]] — Hermes → Yule 흡수 작업
- [[task-log-harness-issue-48]] — Harness 패턴 적용 (#48)

## 운영 규칙

1. 본문 표준 5 섹션: 목표 / 진행 (시계열 또는 commit 별) / 결정 요약
   (`decisions/` cross-link) / 비결정 / 다음 행동.
2. 같은 사이클의 다른 task-log 가 있으면 frontmatter `related` 로
   cross-link.
3. 사이클 종료 시 본 폴더 README 의 "현재 노트" 목록을 갱신하고 hub
   (`_moc/<scope>.md`) 에 [[link]] 추가.
4. 작업 중 발견한 mistake 는 `hookify` ledger 와 별개로 본 task-log
   에도 기록 (운영자 grep 단위 추적).

## 관련 hub

- [[../_moc/_moc|_moc/ 전체 인덱스]]
- [[../_moc/f15-corporate-structure]]
- [[../_moc/issue-73-tech-lead-runtime]]
- [[../_moc/discord-ci-strategy]]
