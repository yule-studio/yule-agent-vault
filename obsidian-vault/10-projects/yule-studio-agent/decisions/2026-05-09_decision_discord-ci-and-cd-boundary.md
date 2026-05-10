---
title: "Discord CI 첫 도입 결정 — PR 기준 검증과 배포 분리"
kind: decision
session_id: discord-ci-design
project: yule-studio-agent
created_at: 2026-05-09T00:00:00+09:00
agent: engineering-agent/tech-lead
status: decided
related:
  - ../research/2026-05-09_research_discord-ci-strategy.md
  - ../task-logs/2026-05-09_task-log_discord-ci-design.md
tags:
  - decision
  - ci
  - discord
  - github-actions
---

# 목표

Yule Studio Agent 레포에서 첫 CI 를 어떤 기준으로 시작할지 결정한다. 핵심은 "PR 품질 게이트를 만들고, 결과를 Discord 에 명확히 노출" 하는 것이다.

# 결정

| ID | 항목 | 결정 |
| --- | --- | --- |
| D-CI-1 | CI 실행기 | GitHub Actions 채택 |
| D-CI-2 | 첫 workflow 책임 | PR 기준 테스트 + Discord 성공/실패 알림 |
| D-CI-3 | 이벤트 트리거 | `pull_request` (`opened`, `synchronize`, `reopened`, `ready_for_review`) |
| D-CI-4 | push-only 사용 여부 | 비채택 |
| D-CI-5 | 배포 포함 여부 | 이번 단계 비포함 |
| D-CI-6 | workflow 파일명 | `.github/workflows/ci.yml` |
| D-CI-7 | 테스트 명령 | `python3 -m unittest discover -s tests -t .` |
| D-CI-8 | Discord 이미지 제공 방식 | Discord CDN URL 을 GitHub Variables 로 관리 |
| D-CI-9 | 알림 구조 | `test` job + `notify` job 분리 |

# 각 결정의 의미

## D-CI-1 GitHub Actions 채택

현재 저장소는 GitHub 중심으로 운영되고 있고, PR 흐름과 가장 자연스럽게 결합되는 실행기도 GitHub Actions 이다. Jenkins 를 지금 바로 도입하면 유지보수 포인트만 늘어나므로 보류한다.

## D-CI-2 첫 workflow 책임 축소

첫 workflow 는 "검증"과 "가시성"에만 집중한다. 즉:

- 코드는 테스트한다
- 결과는 Discord 에 알린다
- 배포는 하지 않는다

이는 첫 단계에서 실패 원인을 단순화하기 위한 결정이다.

## D-CI-3 pull_request 트리거 채택

PR 단위로 품질 게이트를 만들기 위해 `pull_request` 를 표준 트리거로 쓴다. branch push 자체보다 리뷰 흐름과 잘 맞는다.

## D-CI-4 push-only 비채택

push-only 는 단순하지만 PR 중심 운영과 맞지 않는다. 같은 브랜치에 여러 커밋이 쌓일 때, 지금 필요한 "이 PR 이 현재 통과 상태인가"를 보여주기 어렵다.

## D-CI-5 CI 와 CD 분리

이번 레포는 agent platform 이므로, "PR 이 안전한가"와 "운영에 반영해도 되는가"는 다른 질문이다. 따라서:

- CI = 검증
- CD = 배포

로 분리한다.

## D-CI-6 파일명 표준화

`mail.yml` 대신 `ci.yml` 로 정한다. workflow 목적이 이름에 직접 드러나도록 통일한다.

## D-CI-7 테스트 기준 명확화

현재 레포의 공인 기본 테스트 명령은 `docs/testing.md` 기준 `python3 -m unittest discover -s tests -t .` 이다. 첫 CI 는 이 기준을 그대로 사용한다.

## D-CI-8 이미지 변수 사용

공용 success/fail 이미지는 외부 스토리지 대신 Discord CDN URL 을 활용한다.

필요 변수:

- `DISCORD_CI_SUCCESS_IMAGE_URL`
- `DISCORD_CI_FAIL_IMAGE_URL`

필요 secret:

- `DISCORD_CI_WEBHOOK_URL`

## D-CI-9 notify job 분리

테스트와 알림을 분리해:

- 테스트 성공/실패 판별
- 알림 성공/실패 판별

를 독립적으로 추적한다.

# 이번 단계 비범위

다음 항목은 이번 결정에 포함하지 않는다.

1. `main` 머지 후 자동 배포
2. 무중단 배포 전략
3. rollback 자동화
4. 실패 시 agent 자동 수정
5. 멀티 언어 matrix 확장

# 후속 단계

1. PR CI 안정화
2. `push` on `main` smoke workflow 추가 여부 검토
3. branch protection 에 CI 필수화
4. 배포 workflow 분리 설계
5. 다언어 레포에 맞는 공통 CI 템플릿 정리

## 관련 문서

- [[CLAUDE]]
- [[2026-05-09_research_discord-ci-strategy]]
- [[2026-05-09_task-log_discord-ci-design]]
