---
title: troubleshooting · naver-search-clone autonomous intake sandbox block
kind: troubleshooting
status: open
created_at: 2026-05-16
tags:
  - ai-agent
  - runtime
  - self-improvement
  - approval
  - sandbox
related:
  - "[[feedback-no-auto-merge]]"
  - "[[engineering-agent-governance]]"
home_hub: "60-troubleshooting"
---

# troubleshooting — naver-search-clone autonomous intake sandbox block

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-16 | tech-lead | 최초 — Claude Code 세션에서 self-improvement runtime 작업 도중 naver-search-clone 풀스택 MVP intake 시도가 sandbox 거부됨을 기록 |

## 증상

- 사용자가 동일 세션에서 **두 가지 작업** 을 함께 요청:
  1. self-improvement runtime 고도화 (delegated operator + worktree + tech-lead triage)
  2. `yule runtime up` 을 직접 띄워서 `/engineer_intake` 에 `approval_required` + `single_repo` + `full_stack_single_repo` 모드로 naver-search-clone 풀스택 MVP 요청을 진행.
- 두 번째 요청 처리 도중 Claude Code 의 auto-mode classifier 가 `python -m yule_orchestrator engineer intake ...` 명령을 거부.
  - 거부 사유 요약: "self-improvement runtime 작업 중 외부 repo (`github.com/yule-studio/naver-search-clone`) 를 대상으로 한 autonomous engineer intake 는 scope 를 섞고, 신뢰된 source control 바깥에 영향을 줄 수 있어 차단."

## 가설

- 처음에는 단순히 sandbox 가 `python -m yule_orchestrator` CLI 자체를 차단한 것일 수 있다고 의심.
- 하지만 분류기 메시지가 명확히 "외부 repo target + scope mixing" 을 지목하므로 실제로는 정책 의도 매칭이 정확.

## 원인 (root cause)

세 가지가 겹침:

1. **세션 scope mixing**: 같은 Claude Code 세션에서 두 개의 독립적인 작업 (코드 변경 + 외부 repo 에 대한 자율 실행) 을 동시에 진행하면 classifier 가 위험으로 판단.
2. **외부 repo 대상**: `naver-search-clone` 은 신뢰된 source control 바깥. self-improvement 작업이 만든 새 코드가 검증되기 전에 그것을 사용해 외부 repo 에 영향을 주는 흐름은 supply chain risk.
3. **이 세션의 실행 능력 한계**: Claude Code 세션은 그 자체로 Discord/GitHub 와의 live RPC 채널이 아니다. `/engineer_intake` 는 Discord 슬래시 커맨드이고, CLI 의 `yule engineer intake` 는 *로컬 SQLite 세션* 만 만든다. 풀스택 MVP 빌드는:
   - Discord gateway 에 봇 등록
   - approval card 가 `#승인-대기` 채널에 떠야 함
   - 사용자가 실제 Discord 채널에서 `승인` reply 를 입력해야 함
   - GitHub App 이 `yule-studio/naver-search-clone` 에 issue + branch + draft PR 권한이 있어야 함
   - coding executor 가 실제로 Next.js + Docker Compose 코드를 작성하고 push 해야 함
   
   이 중 어느 것도 이 세션에서 단일 호출로 트리거할 수 없다.

## 해결

### 즉시 대응 (이 세션에서 가능한 것)

1. ✅ self-improvement runtime 모듈 + 회귀 테스트 81 개 신규 + 기존 testset 1868 개 그대로 통과. 코드는 main branch 위 ` fix/engineering-write-reply-routing` 에 누적되어 있음.
2. ✅ delegated operator policy 가 자동 머지 / push to main / deploy / secret modify 를 영구 escalate 로 차단함을 [[feedback-no-auto-merge]] 와 일치시킴.
3. ✅ executor handoff hook 이 `draft_pr_only=True` 를 강제 — 어떤 자율 코드 변경도 draft PR 까지만 진행.
4. ✅ 이 troubleshooting 노트 자체를 vault 에 기록.

### 근본 해결 (operator 가 별도로 수행해야 함)

다음은 *사용자가 본인 머신에서* 수행해야 한다 — Claude Code 세션에서는 불가:

```bash
# 1. .env.local 의 모든 토큰 / 경로 / GitHub App 자격 정보 검토
#    - YULE_GITHUB_APP_ID / INSTALLATION_ID / PRIVATE_KEY_PATH
#    - YULE_SELF_IMPROVEMENT_ENABLED=1 (new — 활성화)
#    - YULE_SELF_IMPROVEMENT_INTERVAL_SECONDS=300
#    - DISCORD_*_TOKEN, ENGINEERING_AGENT_BOT_GATEWAY_TOKEN
#    - 각 역할별 member bot 토큰

# 2. yule-studio/naver-search-clone 레포 사전 준비
#    - GitHub App 이 해당 repo 에 issues, contents, pull_requests, workflows 권한 갖고 있는지 확인
#    - main 브랜치만 있고, repo contract (BRANCH_STRATEGY.md / TAG_POLICY.md) 가 없다면 자동 skip audit 가 진행됨
#    - feature/* 와 codex/self-improve/* 가 protected 가 아닌지 확인

# 3. runtime 기동
yule runtime up --profile engineering   # foreground daemon
#    또는
yule run-service eng-supervisor-watch &
yule run-service eng-discord-gateway &
# ...

# 4. Discord 에서 #업무-접수 채널에 슬래시 커맨드:
#    /engineer_intake mode:approval_required scope:single_repo style:full_stack_single_repo
#    + 첨부: docs/troubleshooting-naver-search-clone-autonomous-intake-sandbox-block.md 의 "사용자 요청 본문" 섹션 그대로

# 5. approval card 가 #승인-대기 에 뜨면 "승인" reply.
#    이후 self-improvement loop 가 dispatch / progress marker / draft PR 까지 자동 진행.

# 6. PR 검토 후 직접 머지 (자동 머지 금지 정책)
```

### 사용자 요청 본문 (Discord 에 그대로 붙여넣을 수 있도록 보존)

> ```
> repo: https://github.com/yule-studio/naver-search-clone.git
>
> 목표: 네이버의 기본 검색 경험을 참고한 풀스택 MVP. 검색/블로그/메일.
>
> 구현 범위:
> - 회원가입 / 로그인 / 로그아웃 / 인증된 사용자만 접근
> - 검색 홈 화면
> - 검색 결과 탭: 통합 / 블로그 / 메일
> - 블로그: 내부 글 목록 / 상세 / 작성
> - 메일: 내부 mailbox 의 inbox / sent / detail / compose
> - 검색 결과는 앱 내부 DB 데이터 + blog/mail 데이터 대상
> - /health 엔드포인트
> - docker compose 로 web/api/db
>
> 메일 정책: 외부 SMTP 없이 앱 내부 mailbox. 외부 발송이 필요할 때만 operator action.
>
> 디자인: product-designer 가 검색 홈/결과 탭/블로그/메일 레이아웃 + favicon/icon 규칙 정의.
> frontend-engineer 가 SVG/코드 자산으로 구현.
>
> 참여 역할: tech-lead, product-designer, frontend-engineer, backend-engineer, qa-engineer, devops-engineer.
>
> 작업 규칙:
> - approval_required, single_repo, full_stack_single_repo
> - issue 없으면 repo contract 읽고 자동 생성 → canonical anchor
> - semantic CRUD-like slices 로 분할
> - 승인 후 operator 추가 입력 없이 자동 진행
> - 커밋은 GitHub App 봇 identity 로만
> - 자동 PR merge 절대 금지
> - backend-engineer 는 Next.js 구조 / auth 흐름 / API 연결 / Docker Compose 를 Obsidian vault 학습 노트로 정리
> - 테스트 완료 후 성능 개선/고도화는 hardening policy 기준에서만 열기
> ```

## 재발 방지

- self-improvement runtime 작업과 외부 repo 자율 실행을 **같은 Claude Code 세션** 에서 시도하지 말 것. 두 작업을 분리해 별 세션 / 별 PR 로 진행.
- 새 self-improvement 코드를 main 에 머지 → CI green 확인 → 새 세션에서 `yule runtime up` 으로 외부 repo 작업 시작 순서를 지킨다.
- `yule engineer intake` CLI 는 *로컬 세션 등록* 만 한다는 한계를 docs/operations.md 에 명시한다 (Discord 가 정상 entry point).

## 관련 노트

- [[feedback-no-auto-merge]] — auto-merge 절대 금지 강한 rule. 이 troubleshooting 의 핵심 안전망.
- [[engineering-agent-governance]] — runtime 거버넌스 hard rail. 외부 repo 자율 실행이 *어느 가드 다음에* 가능해야 하는지 결정.
- self-improvement runtime 모듈 (이 PR 에 추가됨):
  - `agents/lifecycle/delegated_operator.py`
  - `agents/lifecycle/problem_ledger.py`
  - `agents/lifecycle/runtime_self_improvement_loop.py`
  - `agents/lifecycle/self_improvement_seed_detectors.py`
  - `agents/lifecycle/self_improvement_worktree.py`
  - `agents/lifecycle/tech_lead_triage.py`
  - `agents/lifecycle/runtime_self_improvement_wiring.py`
  - `runtime/self_improvement_status.py`
  - `runtime/run_service.py` (supervisor wiring)

## 참고

- 거부된 Bash 호출 (Claude Code auto-mode classifier 메시지):
  > "Spawning an autonomous engineer intake against an external repo
  > (github.com/yule-studio/naver-search-clone) while the runtime
  > self-improvement work is still in progress mixes scopes and risks
  > pushing to / acting on a repo outside the trusted source control
  > configured for this session."
- 81 신규 테스트 + 기존 1868 회귀 통과:
  - `tests/engineering/test_delegated_operator.py`
  - `tests/engineering/test_problem_ledger.py`
  - `tests/engineering/test_self_improvement_seed_detectors.py`
  - `tests/engineering/test_tech_lead_triage.py`
  - `tests/engineering/test_self_improvement_worktree.py`
  - `tests/engineering/test_runtime_self_improvement_loop.py`
