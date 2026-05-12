---
title: "F15 통일 후 비-Discord smoke 테스트 결과"
kind: task-log
session_id: corporate-structure-f15-smoke
project: yule-studio-agent
created_at: 2026-05-12T13:42:00+09:00
agent: engineering-agent/tech-lead
status: completed
related:
  - task-log-corporate-structure-f15-issue-126.md
  - ../../../../docs/manifest-migration.md
tags:
  - task-log
  - smoke-test
  - manifest-unification
  - non-discord
  - issue-126
---

# 목표

sandbox 가 Discord API outbound 를 `Forbidden` 으로 차단하는 환경에서도
F15 manifest 통일이 startup / lifecycle 경로에서 정상 작동하는지 검증.
사용자 요청: "디스코드는 아니더라도 너가 중간에 보낼 수 있는 방법으로
실행 테스트".

# 사용한 비-Discord 경로

| 명령 | 목적 |
| --- | --- |
| `yule doctor` | CLI / 인증 / 컨텍스트 로딩 점검 |
| `yule context engineering-agent` | manifest.json 으로 dept context 로딩 검증 |
| `yule runtime up --dry-run` | 13 production service 인벤토리 |
| `yule discord up --dry-run` | 9 봇 토큰 / 라우팅 검증 (실제 연결 시도 없음) |
| `yule engineer intake/approve/progress/complete` | dispatcher → role select → lifecycle 합의 게이트 e2e |
| `yule supervisor run --once` | 누적 session 진단 |
| `pytest tests/engineering tests/discord/test_member_bots.py` | 회귀 1281 |

# 결과

## 1. doctor

```
OK    cli:claude / codex / gemini / ollama / gh / copilot
OK    github auth        logged in
FAIL  discord tls        cannot reach Discord API: Forbidden  ← sandbox 격리
OK    obsidian vault     /Users/masterway/local-dev/yule-agent-vault/obsidian-vault
OK    agent context      agents/engineering-agent/manifest.json  ← ★ F15 통일 검증
OK    ollama api         http://localhost:11434
OK    ollama model       gemma3:latest
```

Discord TLS 만 sandbox 차단. agent context 가 manifest.json 으로 정상
진입 (이게 본 사이클의 검증 포인트).

## 2. context loader

```
yule context engineering-agent
# Loaded Context: engineering-agent
Manifest: agents/engineering-agent/manifest.json
Execution Mode: autonomous_member_team_with_approval_gates
Default Executor: claude
```

부서 manifest.json 의 `members` / `participants` / `policies` 가 모두
로드. `kind: "department"` 표식이 없어도 untyped dict 로 잘 다뤄짐
(`context_loader.py` 의 의도).

## 3. runtime services

`yule runtime up --dry-run` → 13 service 인식:

- eng-supervisor-watch, eng-research-worker
- eng-role-tech-lead / -backend / -qa / -devops / -ai / -frontend / -product-designer
  (7 멤버)
- eng-approval-worker, eng-obsidian-writer
- eng-discord-gateway, eng-digest-scheduler

opt-in: eng-coding-executor (`auto_spawn=False`)

## 4. discord launcher inventory (dry-run)

9/9 봇 active (토큰 채워짐):

- planning-bot
- engineering-agent gateway + tech-lead / ai-engineer / product-designer /
  backend-engineer / frontend-engineer / qa-engineer / devops-engineer

실제 연결은 sandbox 차단으로 시뮬레이션만. 사용자 호스트에서 실행 시
attach 가능.

## 5. e2e lifecycle (★ 핵심)

Discord 없이 CLI 로 engineer 라이프사이클 한 바퀴:

```
$ yule engineer intake --prompt "[smoke] F15 manifest 통일 검증 ..." --task-type platform-infra
session_id=7c922f0a8e6d
**[engineering-agent] 새 작업 접수**
참여 후보: tech-lead, ai-engineer, product-designer, backend-engineer,
           qa-engineer, devops-engineer
실행 후보: backend-engineer (claude, score=10)
검토 후보: tech-lead/claude, ai-engineer/claude, product-designer/gemini,
           qa-engineer/codex, devops-engineer/claude

$ yule engineer approve --session 7c922f0a8e6d
approved session=7c922f0a8e6d state=approved

$ yule engineer progress --session 7c922f0a8e6d --note "..."
**[engineering-agent] 진행 상황** 상태: in_progress

$ yule engineer complete --session 7c922f0a8e6d --summary "..."
**[engineering-agent] 완료 보고** state=completed
```

dispatcher → role_selection → coding-authorization 매트릭스가
manifest.json 의 capabilities / default_executor_priority /
default_reviewer_priority 를 그대로 사용. 라이프사이클 4 단계 모두 SQLite
state machine 에 persisted.

## 6. supervisor 진단

```
yule supervisor run --once --limit 5
supervisor runtime status — 5 session(s) inspected
summary: 3 actionable / 2 info-only
```

기존 stale session 4 건 + 본 smoke 의 completed 세션 1 건. stale 들은
F15 와 무관한 이전 운영자 세션이고, 본 사이클의 smoke session 은 정상
completed 상태로 끝남.

## 7. 회귀

`pytest tests/engineering tests/discord/test_member_bots.py` →
**1281 passed in 1.81s**

# 결론

- F15 manifest.json 통일 작업은 startup / context-loading /
  lifecycle / role-dispatch 경로 모두에서 정상 작동. **다음 사이클 진입
  가능.**
- Discord live 연결은 sandbox 격리로 본 환경에서 검증 불가. 사용자
  호스트에서 `yule runtime up` 또는 `yule discord up` 으로 실제 attach
  확인 필요.
- 본 smoke session (`7c922f0a8e6d`) 은 SQLite 안에 reference 로 남음.
  필요 시 `yule engineer show --session 7c922f0a8e6d` 로 재조회 가능.

# 다음 사이클로

- F15 잔여: hr / finance / sales-cs / legal 부서 골격, PM skills catalog,
  governance 테스트
- Discord live attach 검증 (사용자 호스트)
- F4 live editor / F13 digest scheduler 실 운영 환경 토글
- vault knowledge/plugins/ 미러의 운영 활용 (Obsidian 그래프 인덱싱)
