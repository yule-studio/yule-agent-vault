---
title: "Hermes Agent → Yule 흡수 결정 (issue #59)"
topic: hermes-yule-integration-decisions
kind: decision
project: yule-studio-agent
session_id: issue-59
issue_number: 59
issue_url: https://github.com/yule-studio/yule-studio-agent/issues/59
external_reference: https://github.com/NousResearch/hermes-agent
related_research: 10-projects/yule-studio-agent/research/2026-05-08_hermes-agent-architecture-deep-dive.md
related_task_log: 10-projects/yule-studio-agent/task-logs/2026-05-08_59-hermes-tech-lead.md
actor_role: engineering-agent/tech-lead
status: decided
created_at: 2026-05-08
---

# Hermes Agent → Yule 흡수 결정 (issue #59)

본 결정 노트는 [Hermes architecture deep-dive](../research/2026-05-08_hermes-agent-architecture-deep-dive.md) 결과를 받아 **Yule 에 무엇을, 어디에, 왜 도입하는지** 를 단일 source of truth 로 정한다. 이후 작성되는 정책 5 종 문서가 본 결정의 구현물이다.

## 목표

- 8 개 영역별로 **도입 / 부분 도입 / 비도입** 을 commit-가능한 결정으로 확정.
- 각 결정의 **WHY (이유)** 와 **WHERE (구현 위치)** 와 **WHAT-NOT (도입하지 않는 것)** 을 같이 적는다.
- #25 / #48 와의 충돌 표면을 명시한다.

## 현재 Yule 기준선

- single-operator + Discord-only + lifecycle 기반 audit-traceable 운영.
- session.extra 영속화 + Obsidian export 가 자산화의 절반을 이미 한다.
- 부족한 영역 — recall / context 압축 / scheduled work / 회고 자동화.

자세한 기준선은 research note §"현재 Yule 기준선" 참조.

## 참고한 외부 레퍼런스

- https://github.com/NousResearch/hermes-agent (full repo survey)
- 분석 module: `gateway/` / `agent/memory_manager.py` / `agent/curator.py` / `agent/context_compressor.py` / `agent/insights.py` / `cron/scheduler.py` / AGENTS.md / README.md

## 결정 요약 (8 영역)

| 영역 | 결정 | 산출물 |
|---|---|---|
| 1. messaging gateway | **비도입** | (없음) |
| 2. persistent memory | **부분 도입** | `memory-policy.md` 신규 |
| 3. cross-session recall | **부분 도입** | `recall-policy.md` 신규 |
| 4. context / trajectory compression | **부분 도입** | `context-compression.md` 신규 |
| 5. skill self-improvement | **부분 도입** | `self-improvement-flow.md` 신규 |
| 6. scheduled automations | **부분 도입** | `scheduled-automation.md` 신규 |
| 7. subagent / parallel workstream | **비도입** | (없음) |
| 8. workspace migration | **비도입** | (없음) |

## 결정 별 상세

### D-1. messaging gateway — 비도입

**Hermes** — `gateway/` 단일 process 가 6 개 platform(Telegram/Discord/Slack/WhatsApp/Signal/CLI) 동시 운영.

**Yule** — Discord 만. `engineering_channel_router.py` 가 4 가지 routing action 분기.

**WHY 비도입**:
- Yule 은 single-operator 환경. 1 인 운영자가 6 개 messenger 를 동시에 들여다볼 시나리오가 없다.
- multi-platform 추상화는 token 표면, secret 표면, 라우팅 복잡도 모두 증가시킨다.
- Hermes 의 `platform_registry` 가 하는 추상화 정도는 GitHub WorkOS 의 `SourceKind = github | discord` 가 이미 비슷한 수준으로 한다.

**WHAT-NOT** (비도입 보장):
- Telegram / Slack / WhatsApp / Signal / CLI 어댑터 추가 없음.
- `discord/engineering_channel_router.py` 본문 수정 없음 (#25 충돌 회피).

### D-2. persistent memory — 부분 도입

**Hermes** — `MemoryProvider` 추상화 위에서 `prefetch()` / `sync_turn()` / `handle_tool_call()`. write trigger = (a) 매 turn 끝, (b) 명시 memory tool 호출. periodic nudge ("이거 기억할 가치 있음") 신호.

**Yule 현재** — SQLite memory index + Obsidian vault. write trigger = lifecycle gate 통과 산출물만.

**WHAT 도입**:
- "기억 후보 hint" 정책. lifecycle 산출물(synthesis / work_report / decision) 중 **재사용성이 높음** 표식이 붙은 것은 추가 trigger 없이 knowledge note 자동 생성 후보로 잡는다.
- 표식 기준: tag/status 메타 (예: `reusable=true`, `status=decided`) 가 이미 session.extra 에 있음 — 그것을 *recall 검색 우선순위 boost* 로 정책화.
- 새 모듈 추가 없음 — 정책 문서만.

**WHY 부분 도입**:
- 자율 memory write 는 Yule 의 single-operator 안전 정책에 위배. lifecycle gate 우회 금지.
- 다만 "이미 통과한 산출물 중 어느 것이 향후 작업의 1 차 input 인가" 의 hint 는 운영자 보고에 유용 — supervisor / status diagnostic 이 활용 가능.

**WHERE**:
- `policies/runtime/agents/engineering-agent/memory-policy.md` 신규.
- agent.json `policies` 배열에 등록.

**WHAT-NOT**:
- 자율 memory write 표면 추가 안 함.
- `agents/memory.py` 본문 수정 없음.
- `agents/lifecycle_persistence.py` 본문 수정 없음 (#25 충돌 회피).

### D-3. cross-session recall — 부분 도입

**Hermes** — `agent/insights.py` 가 `generate(days=N)` 으로 일자별 sessions / messages / tools / 토큰 / cost 보고.

**Yule 현재** — recall 은 thread 단위 topic ledger + `yule memory search` (검색).

**WHAT 도입**:
- "지난 N 일 회고 보고서" 정책 — 어떤 데이터가 어디에 있는지 운영자가 한 곳에서 본다.
- 데이터 source: `task-logs/` + `decisions/` + `research/` + supervisor `agent_ops_audit` 키.
- 정책 문서가 보고서 생성 흐름을 정의 — 코드 generation 은 후속 milestone (`yule insights --days 7` 같은 CLI 후보 명시).

**WHY 부분 도입**:
- supervisor 의 single-session 진단은 잘 되지만 cross-session 회고가 비어 있음.
- 시니어 개발자는 "어제 무슨 결정 했지?" 를 5초 안에 답할 수 있어야 — Obsidian 폴더 grep 하는 것보단 보고서 1 개로 보는 게 자산 운용 효율이 높다.

**WHERE**:
- `policies/runtime/agents/engineering-agent/recall-policy.md` 신규.

**WHAT-NOT**:
- 실제 `cli/insights.py` 코드 작성 안 함 (이번 phase 는 정책까지).
- `agents/memory.py` retrieval API 수정 없음.

### D-4. context / trajectory compression — 부분 도입

**Hermes** — `agent/context_compressor.py`. `threshold_percent=50%` (context window 의 절반), `protect_first_n=3` (첫 N 메시지 보호), `summary_target_ratio=20%` (압축 후 비율), tool output 은 `[terminal] ran X → exit 0, 47 lines` 형태로 압축. 사용자 명령 `/compress <topic>` 도 지원.

**Yule 현재** — 명시 압축 없음. role take 만 `output_sections` 로 짧게 강제. LLM runner(M11b) 활성 시 thread 가 길어지면 동일 문제 발생 가능.

**WHAT 도입**:
- 압축 정책 — token threshold / 보호 영역 / tool output 요약 형태.
- 호출 시점:
  - role-runner dispatch 입력에서 prior turns 가 임계 초과 시.
  - knowledge writer 호출 시 long body 자동 요약 (이미 부분적으로 됨 — 정책으로 끌어올림).
  - work_report final 생성 시 long executive_summary 자동 압축.
- 보호 영역 정의 — *원문 prompt + lifecycle decision* 은 절대 압축 안 함 (audit traceability).

**WHY 부분 도입**:
- LLM runner 가 활성화되는 환경에서 token cost 안전판 필요.
- Yule 의 lifecycle audit 은 *원문 prompt 보존* 이 핵심 — 그 부분은 절대 압축에서 제외.

**WHERE**:
- `policies/runtime/agents/engineering-agent/context-compression.md` 신규.

**WHAT-NOT**:
- 원문 prompt / decision / synthesis 본문 압축 금지.
- 자동 압축 코드 작성 안 함 (정책까지).
- knowledge_writer 본문 수정 없음 (M9/M10 와 충돌 회피).

### D-5. skill self-improvement — 부분 도입

**Hermes** — `agent/curator.py` 가 idle 시 background 로 skill state transition (active → stale → archived) + LLM consolidation.

**Yule 현재** — `agents/lifecycle/self_improvement.py` skeleton + `retrospectives/` Obsidian 폴더. 자동화 흐름 없음.

**WHAT 도입**:
- session 종료 시 retrospective note 자동 생성 후보 알림 정책.
- retrospective 가 **다음 작업의 input** 으로 들어가는 trigger 정의 — `yule engineer intake` 가 동일 영역 retrospective 가 있으면 자동 첨부 후보.
- 정책 문서가 회고 → 다음 input 의 lifecycle 위치를 명시.

**WHY 부분 도입**:
- 자동 LLM consolidation 은 명시적 단일 source-of-truth 가 깨질 위험. Yule 패턴은 운영자 명시 갱신.
- 단 "회고 자산이 다음 작업에 자동 input 으로 들어간다" 는 가치 큼 — 정책으로 박는다.

**WHERE**:
- `policies/runtime/agents/engineering-agent/self-improvement-flow.md` 신규.

**WHAT-NOT**:
- 자동 LLM-driven skill consolidation 안 함.
- `agents/lifecycle/self_improvement.py` 코드 수정 없음 (이번 phase 정책까지).
- `role_profiles_data.py` 자동 갱신 흐름 없음.

### D-6. scheduled automations — 부분 도입

**Hermes** — `cron/scheduler.py` + `jobs.json` (`schedule` / `deliver` / `script` / `prompt` / `model` / `provider`). 60s `tick()`. delivery routing `_resolve_delivery_targets()`.

**Yule 현재** — `discord/bot.py` 안 일 단위 loop 만. 일반 반복 작업 흐름 없음. systemd `yule run-service` 가 worker 단위 always-on.

**WHAT 도입**:
- 반복 작업 운영 정책 — job storage 형태 / 트리거 / safety-guard.
- safety-guard 핵심: (a) secret 미주입 — scheduled job 은 자기 secret 을 들고 있지 않고 인증된 Yule worker 환경에서만 실행, (b) destructive 작업 금지 (push / merge / deploy / rm -rf), (c) 운영자 알림 채널 명시 (Discord 단일 — 새 platform 추가 없음).
- 정책 문서가 향후 `cli/yule scheduler` 또는 systemd timer 어느 쪽으로 갈지 후보 명시.

**WHY 부분 도입**:
- "월요일 의존성 점검" / "주 1 회 backlog 회고 회상" 같은 반복 작업이 운영자 손길 없이 굴러가야 자산 운용 효율 ↑.
- 단 secret / 외부 API 호출 표면 확장은 보수적으로.

**WHERE**:
- `policies/runtime/agents/engineering-agent/scheduled-automation.md` 신규.

**WHAT-NOT**:
- `cron/` 또는 `agents/lifecycle/scheduler.py` 코드 작성 안 함 (정책까지).
- 새 platform / Telegram / Slack 추가 없음.
- secret rotation 정책 변경 없음 — 기존 `.env.local` 격리 그대로.

### D-7. subagent / parallel workstream — 비도입

**Hermes** — `tools/delegate_tool.py` 동기 in-process subagent fork. leaf vs orchestrator role. depth/concurrent cap.

**Yule 현재** — member-bot × 7 (독립 프로세스) + worker pool (research/role/approval/obsidian-writer). single-executor 정책으로 parallel write 거부.

**WHY 비도입**:
- Hermes subagent 는 LLM agent 가 자기 자신을 위임 호출. Yule 은 task 의 도메인 책임이 명시 분리(7 role) 된 부서 모델. **다른 문제를 푼다**.
- 무비판 도입 시 Yule 의 single-executor 안전 정책과 충돌. parallel write 가 audit-traceable 하지 않은 채로 깔리면 운영 사고 위험.
- Hermes 의 안전 가드 (depth=2, concurrent=3) 같은 패턴은 Yule worker queue 길이 + heartbeat + circuit breaker 가 이미 같은 역할.

**WHAT-NOT**:
- subagent / `delegate_task` tool 추가 없음.
- worker pool 본문 수정 없음 (#25 / #48 충돌 회피).
- single-executor 정책 변경 없음.

### D-8. workspace migration — 비도입

**Hermes** — OpenClaw → Hermes settings/memories/skills/keys/personas/allowlists 자동 import.

**Yule 현재** — 해당 use case 없음.

**WHY 비도입**:
- Yule 사용자가 OpenClaw 같은 다른 LLM agent 에서 마이그레이션할 시나리오 = 0.
- 비대칭 흡수 — 기능 자체에 가치 없음.

**WHAT-NOT**:
- migration 코드 / 정책 / 문서 추가 없음.

## 도입한 부분 (본 결정 기준)

5 개 정책 신규:
1. `policies/runtime/agents/engineering-agent/memory-policy.md` (D-2)
2. `policies/runtime/agents/engineering-agent/recall-policy.md` (D-3)
3. `policies/runtime/agents/engineering-agent/context-compression.md` (D-4)
4. `policies/runtime/agents/engineering-agent/self-improvement-flow.md` (D-5)
5. `policies/runtime/agents/engineering-agent/scheduled-automation.md` (D-6)

`agents/engineering-agent/agent.json` 의 `policies` 배열에 5 개 정책 경로 추가.

## 보류 / 비도입 부분

- D-1, D-7, D-8 (3 개 영역) — 이유는 위 §"WHAT-NOT" 절에 단일 source.

## 왜 실제 시니어 개발팀형 회사 구현에 필요한지

Yule 의 lifecycle / Obsidian / topic ledger 가 이미 *자산화의 절반* 을 한다 — 작업 산출물이 SQLite + vault 에 audit-traceable 하게 들어간다.

부족한 *자산을 다시 꺼내 쓰는 흐름*:
- **memory recall hint** (D-2) — 무엇이 다음 작업의 1 차 input 인가
- **cross-session 회고 보고** (D-3) — 어제/지난 주에 무슨 결정을 내렸나
- **context 압축** (D-4) — 자산이 누적될 때 비용 통제
- **회고 자동화** (D-5) — 회고가 다음 input 으로 자동 흐르게
- **반복 작업 운영** (D-6) — 정기 점검이 운영자 손 없이 굴러가게

이 5 개가 정책으로 명시되면 시니어 개발팀형 운영 — 한 번 결정한 것은 다시 결정하지 않고, 한 번 본 자료는 다시 찾지 않고, 한 번 깨진 회귀는 다시 깨지지 않는 흐름 — 의 토대가 된다.

## 구현 위치

`policies/runtime/agents/engineering-agent/` 트리에 5 개 정책 신규 + `agents/engineering-agent/agent.json` `policies` 배열 갱신. 코드 변경은 본 phase 범위 밖 — 각 정책은 후속 milestone 코드의 anchor 역할.

## 리스크와 다음 액션

- 리스크 1 — 정책만 있고 코드 없음 상태가 길어지면 정책 자체가 stale 화. 각 정책 마지막에 *후속 코드 milestone 후보* 를 명시.
- 리스크 2 — 다른 worktree (#25 / #48) 가 같은 정책 영역을 건드릴 수 있음. 본 phase 정책은 *Hermes 흡수 영역에 한정* — 일반 lifecycle / 팀 아키텍처 정책 본문 수정 없음.
- 리스크 3 — `agent.json` `policies` 배열 변경이 다른 worktree 와 충돌 가능. 본 phase 는 5 줄 추가 only — git 충돌 시 머지 키 단순.

다음 액션:
1. 5 개 정책 문서 작성
2. `agent.json` `policies` 배열 갱신
3. 테스트 / self-check
4. progress comment + draft PR
