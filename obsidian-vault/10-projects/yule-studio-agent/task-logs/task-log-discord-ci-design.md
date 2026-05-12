---
title: "Discord CI 설계 작업 로그"
kind: task-log
session_id: discord-ci-design
project: yule-studio-agent
created_at: 2026-05-09T00:00:00+09:00
status: in-progress
agent: engineering-agent/tech-lead
tags:
  - task-log
  - ci
  - discord
  - github-actions
---

# 목표

PR 기준 Python CI 와 Discord 성공/실패 알림의 설계 과정을 기록한다. 이 노트는 "왜 이렇게 정했는지"보다 "오늘 무엇을 확인했고, 어떤 입력값이 준비됐고, 다음에 뭘 구현하면 되는지"를 남기는 working log 이다.

# 오늘 확인한 입력값 / 전제

- Discord Webhook 연결은 이미 완료
- success / fail 이미지는 Discord 채널에 업로드 완료
- 이미지 URL 은 GitHub Variables 로 정리:
  - `DISCORD_CI_SUCCESS_IMAGE_URL`
  - `DISCORD_CI_FAIL_IMAGE_URL`
- Discord Webhook URL 은 GitHub Secret 으로 사용:
  - `DISCORD_CI_WEBHOOK_URL`

# 오늘 정리한 논점

## 1. GitHub App Webhook 과 Discord Webhook 은 다르다

- GitHub App Webhook: GitHub 이벤트를 받는 입력 채널
- Discord Webhook: Discord 로 메시지를 보내는 출력 채널

이번 CI 알림은 후자다. 따라서 GitHub App webhook 설정과 CI Discord 알림 설정은 분리해서 본다.

## 2. CI 는 Actions, 배포는 나중

지금 필요한 것은:

- 테스트가 통과했는지
- PR 기준으로 알림이 잘 가는지

이지, 아직 무중단 배포가 아니다. 그래서 CI 와 CD 를 나눈다.

## 3. push-only 보다 pull_request 가 적합

오늘 검토 결과, push-only 는 단순하지만 PR 중심 운영과 맞지 않는다. 첫 CI 는 `pull_request` 이벤트로 시작한다.

## 4. 이미지 호스팅은 Discord CDN 재사용

Pages, S3, assets 레포 없이도 빠르게 시작하려면 Discord 업로드 파일 URL 을 그대로 쓰는 것이 가장 실용적이다.

# 추천 workflow 구조

```text
test job
  ├─ checkout
  ├─ setup python
  ├─ install deps
  └─ run unittest

notify job
  ├─ if success -> Discord success embed + success image
  └─ if failure -> Discord failure embed + fail image
```

# 현재 기준 명령

```bash
python -m pip install --upgrade pip setuptools wheel
python -m pip install -e .
python3 -m unittest discover -s tests -t .
```

# 앞으로 다양한 언어로 갈 때 메모

이번에는 Python 하나만 다루지만, 구조는 언어 공통 템플릿으로 유지한다.

- Python: `setup-python`, `pip install -e .`, `unittest`
- Node: `setup-node`, `npm ci`, `npm test`
- Java/Gradle: `setup-java`, `./gradlew test build`
- Go: `setup-go`, `go test ./...`

즉 공통 프레임:

1. trigger
2. setup
3. install
4. test/build
5. notify

이 중 2~4만 언어별로 달라진다.

# 다음 액션

1. `.github/workflows/mail.yml` 대신 `.github/workflows/ci.yml` 로 정리
2. proposed workflow YAML 작성
3. 샘플 PR 로 Discord success/failure 알림 검증
4. 필요하면 이후 `push` on `main` smoke 검토

# 메모

- Discord CDN URL 방식은 빠르지만, 원본 메시지가 삭제되면 이미지가 깨질 수 있다
- 장기적으로 공용 자산이 늘어나면 assets 레포 또는 Pages 를 다시 검토할 수 있다
- 이번 단계에서는 "지금 바로 동작"이 우선이다

## 관련 문서

- [[CLAUDE]]
- [[research-discord-ci-strategy]]
- [[decision-discord-ci-and-cd-boundary]]
