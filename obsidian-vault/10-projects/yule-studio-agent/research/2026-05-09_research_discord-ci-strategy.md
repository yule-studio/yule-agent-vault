---
title: "Discord CI 알림 전략 — GitHub Actions + Discord Webhook + 이미지 URL"
kind: research
session_id: discord-ci-design
project: yule-studio-agent
created_at: 2026-05-09T00:00:00+09:00
agent: engineering-agent/tech-lead
tags:
  - research
  - ci
  - discord
  - github-actions
  - automation
---

# 목표

Yule Studio Agent 레포의 첫 CI 알림 체계를 설계한다. 이번 단계의 목표는 "PR 기준 테스트 성공/실패를 GitHub Actions가 판단하고, Discord 채널에 즉시 보이게 알린다" 이다.

# 현재 기준선

- Python 프로젝트의 기본 테스트 명령은 `python3 -m unittest discover -s tests -t .`
- Discord 운영 규약은 `docs/discord.md`
- 테스트 운영 규약은 `docs/testing.md`
- GitHub App은 issue/PR/worktree/orchestration surface 용도이고, CI 성공/실패 판단은 GitHub Actions가 맡는 편이 자연스럽다
- 성공/실패 이미지는 레포/Pages/S3 대신 Discord 업로드 파일의 CDN URL을 GitHub Variables로 저장한 상태

# 왜 GitHub Actions + Discord Webhook 인가

이번 문제는 "agent orchestration"보다 "코드 검증 결과를 빨리, 확실하게 알려주는 것"이 핵심이다. 따라서:

1. 테스트 실행은 GitHub와 가장 가까운 실행기인 GitHub Actions가 맡는다.
2. 결과 알림은 Discord Webhook으로 바로 보낸다.
3. GitHub App은 PR 본문/코멘트/이슈 작업의 주체로 유지하고, CI와 역할을 섞지 않는다.

이렇게 분리하면 실패 원인도 단순해진다.

- 테스트 실패 = CI 문제
- Discord 메시지 전송 실패 = 알림 문제
- issue/PR 자동화 실패 = GitHub App 문제

# 왜 CI 와 CD 를 분리하는가

이번 레포는 "회사형 에이전트 운영 플랫폼"이므로, PR 단계에서 가장 먼저 필요한 것은 **품질 게이트**다. 배포는 그 다음 문제다.

CI 와 CD 를 분리하는 이유:

1. PR 단계에서는 "머지해도 되는가"만 판별하면 된다.
2. 배포는 `main` 반영 이후의 운영 정책, 무중단 전략, 롤백 전략까지 포함하므로 책임 범위가 더 크다.
3. CI 와 CD 를 한 workflow 안에 섞으면 실패 원인과 운영 권한 경계가 흐려진다.
4. 향후 여러 언어/프로젝트로 확장할 때도 "PR 검증"과 "운영 반영"은 공통적으로 분리되는 편이 유지보수에 유리하다.

이번 단계의 기준:

- CI: `pull_request` 트리거, 테스트, Discord success/failure 알림
- CD: 이번 단계 비범위
- 후속 단계:
  - `push` on `main` smoke
  - 승인된 배포 workflow
  - 무중단 배포 자동화

# 이벤트 트리거 비교

| 트리거 | 장점 | 단점 | 이번 단계 채택 여부 |
| --- | --- | --- | --- |
| `push` | 단순함 | PR 문맥이 약하고 브랜치 푸시마다 시끄러움 | 보류 |
| `pull_request` | PR 중심 검증, 리뷰 흐름과 맞음 | draft/ready 상태 고려 필요 | 채택 |
| `push` on `main` | 머지 후 smoke 용도 좋음 | PR 게이트 대체 불가 | 후속 단계 |
| `workflow_run` | 상위 workflow 후속 연동 가능 | 초기 설계 복잡도 증가 | 후속 단계 |

결론: 이번 첫 CI 는 `pull_request` 를 기준으로 시작하는 것이 가장 적합하다.

# 왜 push-only 가 아닌가

push-only 는 "브랜치에 무언가 올라갔다"는 사실만 잘 보여준다. 그러나 현재 사용자 목표는 "PR 기준으로 테스트가 성공했는지 실패했는지 Discord 에서 즉시 보이는 것"이다. 이 목표에는 `pull_request` 가 더 직접적이다.

push-only 의 문제:

- 같은 PR 에 커밋이 여러 번 올라오면 브랜치 push 맥락만 남고 PR 맥락이 흐려진다
- draft → ready 전환 같은 리뷰 이벤트를 다루기 어렵다
- 앞으로 branch protection 과 결합할 때 PR 상태 중심 모델과 어긋난다

# 이미지 전략

이번 설계에서 success/fail 이미지는 외부 스토리지를 쓰지 않고 Discord CDN URL 을 그대로 재사용한다.

현재 변수명:

- `DISCORD_CI_SUCCESS_IMAGE_URL`
- `DISCORD_CI_FAIL_IMAGE_URL`

이 방식의 장점:

1. S3, GitHub Pages, 별도 assets 레포가 필요 없다.
2. 공용 CI 자산 2개를 빠르게 적용할 수 있다.
3. workflow 에서는 단순히 embed image URL 로만 사용하면 된다.

주의:

- Discord 원본 메시지/파일을 삭제하면 URL 이 깨질 수 있다.
- 장기 운영에서 공용 자산이 늘어나면 별도 assets 저장소나 Pages 로 옮기는 것이 더 안정적일 수 있다.

# Python 기준 첫 CI 설계

첫 workflow 의 책임:

1. PR 이벤트 수신
2. Python 설치
3. 프로젝트 의존성 설치
4. 전체 `unittest` 실행
5. 결과를 Discord 로 전송

기준 명령:

```bash
python -m pip install --upgrade pip setuptools wheel
python -m pip install -e .
python3 -m unittest discover -s tests -t .
```

## notify job 분리 이유

테스트 job 과 알림 job 을 분리하면:

- 테스트 결과를 `needs.test.result` 로 명확히 읽을 수 있다
- 알림 job 은 `if: always()` 로 항상 실행 가능하다
- 테스트 성공/실패와 알림 성공/실패를 따로 디버깅할 수 있다

# 앞으로 여러 언어로 확장할 때의 기준

이번 설계는 Python 전용이지만, 구조는 언어 공통으로 재사용한다.

공통 원칙:

1. workflow 이름은 공통적으로 `CI`
2. PR 트리거는 유지
3. job 은 `test` 와 `notify` 로 분리
4. 언어별 차이는 setup/install/test 단계에서만 분기
5. Discord 알림 포맷은 공통 유지

언어별 확장 예시:

| 언어/스택 | setup | install | test/build |
| --- | --- | --- | --- |
| Python | `actions/setup-python` | `pip install -e .` | `python3 -m unittest ...` |
| Node.js | `actions/setup-node` | `npm ci` | `npm test`, `npm run build` |
| Java / Gradle | `actions/setup-java` | wrapper 사용 | `./gradlew test build` |
| Go | `actions/setup-go` | `go mod download` | `go test ./...` |
| Rust | toolchain setup | `cargo fetch` | `cargo test` |

이때도 기본 프레임은 동일하게 유지하고, 언어별 job template 만 분기하면 된다.

# 추천 파일명과 운영 방식

workflow 파일명은 `.github/workflows/ci.yml` 이 가장 명확하다.

이유:

- `mail.yml` 보다 역할이 직접적이다
- 이후 `deploy.yml`, `smoke.yml`, `release.yml` 과도 구분이 쉽다
- 저장소에 들어오는 새 기여자도 의도를 바로 이해할 수 있다

# 이번 단계의 권장 범위

이번에 바로 하는 것:

1. `.github/workflows/ci.yml`
2. `pull_request` 트리거
3. Python 테스트
4. Discord success/failure 알림
5. 이미지 URL 변수 사용

이번에 하지 않는 것:

1. 자동 배포
2. Jenkins 연동
3. main 머지 즉시 운영 반영
4. 실패 시 agent 자동 수정

# 다음 액션

1. `.github/workflows/ci.yml` 작성
2. `DISCORD_CI_WEBHOOK_URL` secret 확인
3. `DISCORD_CI_SUCCESS_IMAGE_URL`, `DISCORD_CI_FAIL_IMAGE_URL` variable 확인
4. 샘플 PR 로 success/failure 알림 검증
5. 검증 후 branch protection 에 CI 통과 요구 추가 검토

## 관련 문서

- [[CLAUDE]]
- [[2026-05-09_decision_discord-ci-and-cd-boundary]]
- [[2026-05-09_task-log_discord-ci-design]]
