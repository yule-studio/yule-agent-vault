---
title: "Hermes Agent 구조 분석 — Yule 비교 deep-dive"
topic: hermes-architecture-survey
kind: research
project: yule-studio-agent
session_id: issue-59
issue_number: 59
issue_url: https://github.com/yule-studio/yule-studio-agent/issues/59
external_reference: https://github.com/NousResearch/hermes-agent
actor_role: engineering-agent/tech-lead
status: captured
created_at: 2026-05-08
sources:
  - https://github.com/NousResearch/hermes-agent
  - https://raw.githubusercontent.com/NousResearch/hermes-agent/main/README.md
  - https://raw.githubusercontent.com/NousResearch/hermes-agent/main/AGENTS.md
  - https://raw.githubusercontent.com/NousResearch/hermes-agent/main/agent/memory_manager.py
  - https://raw.githubusercontent.com/NousResearch/hermes-agent/main/agent/curator.py
  - https://raw.githubusercontent.com/NousResearch/hermes-agent/main/agent/context_compressor.py
  - https://raw.githubusercontent.com/NousResearch/hermes-agent/main/agent/insights.py
  - https://raw.githubusercontent.com/NousResearch/hermes-agent/main/cron/scheduler.py
agent: engineering-agent/tech-lead
---

# Hermes Agent 구조 분석 — Yule 비교 deep-dive

본 노트는 Hermes Agent (NousResearch/hermes-agent) 의 8 개 영역을 코드 차원에서 들여다보고, 각 영역별로 Yule Studio Agent 의 현재 구현과 어떻게 다른지 정리한다. 이후 작성되는 [decision note](../decisions/decision-hermes-yule-integration.md) 에서 도입/보류 결정의 근거 역할을 한다.

## 목표

- Hermes 의 8 개 핵심 영역(messaging gateway / persistent memory / cross-session recall / context compression / skill self-improvement / scheduled automation / subagent orchestration / workspace migration)을 **모듈/파일 단위**로 매핑.
- Yule 의 동일 책임 모듈 위치를 옆에 두고 **diff 기반** 으로 차이를 적는다.
- "Yule 이 부족한 영역" 을 식별해 decision note 의 적용 후보로 넘긴다.
- "Yule 이 이미 잘 하고 있는 영역" 도 명시 — 무비판 흡수를 막는다.

## 현재 Yule 기준선

### 모듈 레이아웃 (관련 영역)

| 책임 | Yule 위치 | 비고 |
|---|---|---|
| Discord 메시지 라우팅 | `discord/engineering_channel_router.py` | 4 가지 action(`join_existing_work` / `create_new_work` / `ask_for_clarification` / `append_context_only`) |
| member-bot turn | `discord/engineering_team_runtime.py` | role-scoped open-call + research turn |
| session 영속화 | `agents/lifecycle_persistence.py` + SQLite cache | `merge_session_extra` / `to_json_safe` |
| ResearchPack 영속화 | `agents/research_persistence.py` | persist_research_artifacts |
| memory 인덱스 | `agents/memory.py` + SQLite FTS5 | `yule memory reindex` / `yule memory search` |
| Obsidian export | `agents/obsidian_export.py` (string) + `agents/obsidian_writer.py` (IO) + CLI | contract `research-forum-export/v0` |
| knowledge note | `agents/knowledge_writer.py` | M9/M10 phase C — 사람 가독 KB 문서 |
| topic ledger | `agents/lifecycle/research_topic.py` | M9 — duplicate dedup, status transitions |
| role profile | `agents/role_profiles.py` + `role_profiles_data.py` | 7 role + 5 ParticipationLevel + 6 fallback policy |
| role runner dispatch | `agents/runners/role_runner.py` + `runners/bootstrap.py` | M11/M11b — Claude/Codex/Ollama → deterministic |
| daily briefing scheduling | `discord/bot.py` (`_run_daily_briefing_loop` 등) | 부분적 — 일 단위 cron 만 |
| GitHub WorkOS | `agents/github_workos/*` | G1~G6 — issue triage → draft PR |
| coding authorization | `cli/engineer.py` + router coding gate | L0~L3 권한 분리, secret 차단 |
| supervisor / status | `agents/session_status.py` + `cli/supervisor.py` | session 상태 진단 + 운영자 패치 제안 |

### 운영 방식

- **single-operator** 환경. 개인 홈서버에서 1 명이 운영.
- **Discord** 가 전부 — 사용자 입출력 표면. Telegram / WhatsApp / Slack / Signal 은 미지원.
- **single executor** 정책 — 한 작업당 한 role 만 코드 수정 권한. parallel write 거부.
- secret 은 `.env.local` (gitignored) 에만, `.env.example` 에는 placeholder. GitHub App key/token/Authorization 헤더는 audit / log 어디에도 안 나옴 (`redact_secrets`).

## Hermes 8 영역 deep-dive

### 1. Messaging gateway

**Hermes** — `gateway/` 디렉토리 단일 process 가 Telegram / Discord / Slack / WhatsApp / Signal / CLI 6 개 platform 을 동시에 운영. 핵심 파일: `gateway/run.py`, `gateway/session.py`, `gateway/session_context.py`, `gateway/platforms/*`. command: `hermes gateway setup` + `hermes gateway start`. multi-platform routing + sticker/embed 처리 (`sticker_cache.py`, `whatsapp_identity.py`) 까지 포함.

**Yule** — Discord 단일. `discord/engineering_channel_router.py` 가 4 가지 routing action 으로 분기. 다른 platform 은 미지원 (그리고 본 phase 범위 밖).

**Diff 핵심** — Hermes 는 platform 추상화 계층(`platform_registry.py`, `platforms/*`) 을 갖고 platform-agnostic session 을 들고 다닌다. Yule 은 Discord 가 1 급 시민이고 SQLite session 이 thread/forum id 를 직접 들고 있어 다른 platform 으로 옮기려면 schema 변경이 필요.

**Yule 적용 가치** — 낮음. multi-platform 운영은 single-operator 환경에 과한 추상화. Hermes 의 `session_context.py` 가 하는 "platform-agnostic 메시지 모델" 정도는 GitHub WorkOS triage 가 이미 비슷한 추상화(`SourceKind` = `github`/`discord`)를 갖고 있다 — 추가 도입 불필요.

### 2. Persistent memory

**Hermes** — `agent/memory_manager.py` 가 `MemoryProvider` 추상화 위에서 `prefetch()` / `sync_turn()` / `handle_tool_call()` 으로 메모리 read/write 를 라우팅. `OpenClaw` 호환을 위해 `MEMORY.md` / `USER.md` import 도 지원. write trigger 는 (a) 매 turn 끝(`sync_all`), (b) 명시적 memory tool 호출. 동시에 enable 되는 external provider 는 1 개로 제한 — "tool schema bloat" 방지.

**Yule** — SQLite memory index (`agents/memory.py`) + Obsidian vault. write trigger 는 (a) ResearchPack 수집 후 `persist_research_artifacts`, (b) Obsidian sync (lifecycle gate 통과 시), (c) topic ledger 갱신. recall 은 `yule memory search "query"` (FTS5).

**Diff 핵심** — Hermes 는 "agent 가 자기 결정으로" 메모리에 쓴다 (memory tool 호출). Yule 은 "lifecycle gate 가 통과한 산출물만" 영속화. Yule 의 정책이 더 보수적 + 안전 — 자율 write 가 audit-traceable.

**Yule 부족 영역** —
- Hermes 의 `memory_manager.py` 가 갖는 **periodic nudge** ("this might be worth remembering" 신호) 가 Yule 에는 없다. Yule 은 명시 트리거(승인 후 sync)만 갖는다.
- 후보 적용 — knowledge writer 호출 hint: "이 작업은 재사용성이 높음" 표식이 있을 때 자동 trigger 하는 정책. *단 자율 write 는 여전히 lifecycle gate 통과 후로 제한.*

### 3. Cross-session recall

**Hermes** — `agent/insights.py` 가 sessions / messages 테이블 against FTS5 lookup 으로 "what did we work on last week" 형태 보고. `generate(days, source)` / `format_terminal` / `format_gateway`. 토큰 / cost / tool 호출 빈도까지 함께 집계.

**Yule** — recall 은 `agents/lifecycle/research_topic.py` 의 topic ledger 가 thread_id 단위로 한다. cross-session search 는 `yule memory search` (검색만, 활동 보고 없음). `yule supervisor run --once` 가 session 진단 + 운영자 패치 제안.

**Diff 핵심** — Hermes 의 insights 는 정량 분석(token / cost / 빈도). Yule 의 supervisor 는 정성 진단(어디까지 진행 / 무엇이 막혔는지). 둘 다 의미가 있고 충돌하지 않음.

**Yule 부족 영역** —
- "지난 N 일 동안 어떤 작업을 했고 어떤 자료를 봤는지" 의 일자별 보고서가 없다.
- Obsidian task-logs / research / decisions 폴더에 데이터는 다 있지만 한 곳에서 묶어 보여주는 layer 가 없다.
- 후보 적용 — `yule insights --days 7` 같은 운영자 명령. 또는 `agents/lifecycle/recall_index.py` 같은 모듈로 일자별 회고 자동 생성. **이번 phase 는 정책 문서까지만, 코드는 후속.**

### 4. Context / trajectory compression

**Hermes** — `agent/context_compressor.py` 가 token threshold 기반 자동 압축. 핵심 상수: `threshold_percent=50%`, `protect_first_n=3`, `summary_target_ratio=20%`, `_MIN_SUMMARY_TOKENS=2000`, `_SUMMARY_TOKENS_CEILING=12000`. 보호 영역(head + tail) 사이의 middle turn 만 압축, tool output 은 `[terminal] ran npm test → exit 0, 47 lines` 형태로 요약. 사용자 명령 `/compress <topic>` 도 지원.

**Yule** — 없음. role take 는 `RoleProfile.output_sections` 템플릿으로 항상 짧게 강제되지만, 긴 thread 의 context 가 누적되면 LLM runner(M11b dispatch) 가 그대로 보낸다. research budget(provider 호출 수 / 자료 수)으로 *수집* 은 잡지만 *내부 context* 는 잡지 않음.

**Diff 핵심** — Yule 은 작업 1 건 안에서 단발 처리 + Discord forum 댓글 수가 적어 context 폭증이 자주 나지는 않음. 하지만 LLM runner 가 활성화된 환경에서 thread 가 길어지면 동일 문제가 발생.

**Yule 부족 영역** —
- token 누적이 일정 임계를 넘었을 때의 **자동 압축** 정책 없음.
- tool output 압축(예: research provider 가 받은 raw HTML, work_report 의 long body)은 knowledge writer 가 부분적으로 한다 — 공식 정책으로 끌어올릴 가치 있음.
- 후보 적용 — `policies/runtime/agents/engineering-agent/context-compression.md` (정책) + 향후 `agents/lifecycle/context_compressor.py` (코드).

### 5. Skill self-improvement

**Hermes** — `skills/` 디렉토리 = 카테고리별 (apple, devops, github, mcp, productivity 등) 스킬 라이브러리. `agent/curator.py` 가 background 로 idle 시 돌면서 (a) 활성 → stale → archive 자동 transition, (b) 좁은 스킬 → "umbrella" 클래스급 스킬로 LLM 기반 consolidation, (c) cron job reference 자동 갱신, (d) `run.json` + `REPORT.md` 감사 로그. 핵심 원칙: "library of hundreds of narrow skills 는 실패 — class-level instructions + subsections 가 목표".

**Yule** — `agents/lifecycle/self_improvement.py` (skeleton 단계). `policies/runtime/agents/engineering-agent/role-profiles.md` 가 role 정책 source of truth — 운영자가 직접 갱신. 자동 회고 없음.

**Diff 핵심** — Hermes 는 사용자 모름 사이 skill library 정리. Yule 은 모든 변경이 명시적(commit + decision note). Yule 패턴이 single-operator + audit-traceable 환경에 더 맞다 — 자동 consolidation 은 위험(자기가 만든 정책을 자기가 지움).

**Yule 부족 영역** —
- 회고를 자산화하는 흐름은 있지만(retrospective 폴더), 그 자산이 **다음 작업의 input** 으로 자동 들어가는 path 가 약함.
- 후보 적용 — Obsidian retrospective note kind + lifecycle 단계 trigger ("session 종료 시 회고 자동 생성 후보 알림") 의 정책화.

### 6. Scheduled automations

**Hermes** — `cron/scheduler.py`. job storage = `jobs.json` (`id` / `name` / `prompt` / `schedule` / `deliver` / `script` / `skills` / `model` / `provider` / `workdir`). `tick()` 60s 백그라운드 thread, `get_due_jobs()` → `advance_next_run()`. delivery routing = `_resolve_delivery_targets()` 가 `"telegram:chat_id"` 같은 expression 을 platform/chat/thread tuple 로 풀어줌. 자연어 스케줄 parsing 은 보이지 않음 — explicit cron expression 으로 보임.

**Yule** — `discord/bot.py` 안 일 단위 loop 만 (`_run_daily_briefing_loop`, `_run_daily_preparation_loop`, `_run_checkpoint_notification_loop`). 일반화된 cron / scheduled work 흐름 없음. systemd 기반 `yule run-service` 가 worker 별 always-on 운영을 책임짐.

**Diff 핵심** — Yule 은 *worker level* always-on (research-worker / role-worker / approval-worker) 은 갖췄으나, *task level* 반복 작업("매주 월요일 의존성 점검") 은 운영자가 매번 돌려야 함.

**Yule 부족 영역** —
- 반복 작업을 lifecycle 안에 잡는 정책이 없음.
- 후보 적용 — `policies/runtime/agents/engineering-agent/scheduled-automation.md` (정책 / job storage 형태 / 안전 가드: secret 미주입 / Discord 공지 형식). 코드는 이번 phase 범위 밖.

### 7. Subagent / parallel workstream

**Hermes** — `tools/delegate_tool.py` 가 동기 in-process subagent spawn. `delegate_task` tool 호출이 child 를 fork → 부모 block → child summary 반환. role 두 가지: `leaf` (default — `delegate_task`/`memory`/`send_message` 등 거부), `orchestrator` (재귀 spawn 가능). 깊이 cap = `delegation.max_spawn_depth=2`, 동시 cap = `delegation.max_concurrent_children=3`. **durable 하지 않음** — long-running 은 cron 또는 `terminal(background=True)` 사용.

**Yule** — member-bot × 7 (독립 프로세스). research-worker / role-worker / approval-worker / obsidian-writer-worker (worker pool). single-operator 가 한 작업당 한 executor 로 진행 — *parallel write 거부*. 하지만 *parallel read / research collection* 은 role-scoped + multi-provider 로 이미 잘 동작.

**Diff 핵심** — Hermes 의 leaf/orchestrator 모델은 multi-LLM general-purpose agent 가 자기 자신을 위임 호출하는 패턴. Yule 의 role 모델은 task 의 도메인 책임이 명시적으로 나뉜 부서 모델. **둘이 같은 문제를 다르게 푸는 게 아니라 다른 문제를 푼다**.

**Yule 부족 영역** —
- subagent 자체는 도입 불필요.
- 단, Hermes 의 `delegation.max_concurrent_children=3` 같은 **숫자로 박힌 안전 가드** 는 Yule 에도 있는 게 좋음. 현재 Yule 은 worker queue 길이 / heartbeat / circuit breaker 로 잡고 있어 이미 충분 — *추가 작업 없음*.

### 8. Workspace / context migration

**Hermes** — OpenClaw 의 settings / memories / skills / API keys / personas / command allowlists / messaging settings 자동 import. `~/.hermes/skills/openclaw-imports/` 로 들어간 후 운영.

**Yule** — 없음. `.env.local` 수기 작성, vault 수동 마이그레이션.

**Diff 핵심** — Hermes 의 migration 은 동일 카테고리(LLM agent) 에서의 brand switch 시나리오. Yule 에는 해당 use case 자체가 없음.

**Yule 적용 가치** — 0. 비대칭. 도입 안 함.

## Hermes → Yule 적용 매핑표 (research note 시점)

decision note 에서 최종 도입 결정 + 구현 위치까지 확정. 본 표는 분석 결과 1 차 정리.

| 영역 | Hermes 구현 | Yule 현재 | 도입 여부 | 이유 | 구현 위치 (적용 시) | 리스크 |
|---|---|---|---|---|---|---|
| messaging gateway | `gateway/` 단일 process 6 platform | `discord/engineering_channel_router.py` (Discord only) | **보류** | single-operator 환경 — multi-platform 추상화 과함 | (해당 없음) | env 폭증 / secret 표면 확대 |
| persistent memory | `agent/memory_manager.py` + MemoryProvider 추상화 | SQLite + Obsidian + topic ledger | **부분 도입** | "memory write 후보" 인지 trigger 정책만 흡수 | `policies/runtime/agents/engineering-agent/memory-policy.md` 신규 | 자율 write 절대 불가 |
| cross-session recall | `agent/insights.py` (FTS5 + LLM 요약) | `yule memory search` + topic ledger | **부분 도입** | 일자별 회고 보고 정책 추가 | `policies/runtime/agents/engineering-agent/recall-policy.md` 신규 | 운영자 시간 소비 |
| context compression | `agent/context_compressor.py` (50% threshold + 보호 영역) | 없음 | **부분 도입** | 정책 도입(threshold + 보호 영역 + tool output 요약) | `policies/runtime/agents/engineering-agent/context-compression.md` 신규 | 잘못된 압축이 손실로 이어질 수 있음 |
| skill self-improvement | `agent/curator.py` + `skills/` library | `agents/lifecycle/self_improvement.py` skeleton + `retrospectives/` | **부분 도입** | 회고 → 다음 작업 input 자동화 정책 | `policies/runtime/agents/engineering-agent/self-improvement-flow.md` 신규 | 자동 consolidation 위험 — 명시적 트리거만 |
| scheduled automation | `cron/scheduler.py` + `jobs.json` | 일 단위 loop only | **부분 도입** | 반복 작업 정책 + safe-guard | `policies/runtime/agents/engineering-agent/scheduled-automation.md` 신규 | secret 노출 표면 확장 |
| subagent / parallel | `tools/delegate_tool.py` (in-process spawn) | member-bot × 7 + worker pool | **비도입** | 다른 문제. Yule 의 role-scoped + worker pool 로 충분 | (해당 없음) | 무비판 도입 시 single-executor 정책 와해 |
| workspace migration | OpenClaw import | 없음 | **비도입** | use case 부재 | (해당 없음) | (해당 없음) |

## 도입한 부분 (이 노트 기준)

- (research note 단계 — 결정은 decision note 에서 내림. 본 노트는 분석 결과만 담음.)

## 보류/비도입 부분

- 위 매핑표 *비도입* 컬럼.

## 왜 실제 시니어 개발팀형 회사 구현에 필요한지

Hermes 의 흥미로운 부분은 **단발 chat agent 가 아니라 운영 가능한 자산 관리자** 라는 점이다.
- 시니어 개발자는 회의록 / decision log / playbook / runbook 을 *자산* 으로 운영함.
- Hermes 의 curator / memory / insights / cron 이 같이 작동하면 "한 사람이 일 년간 쌓은 작업 자산" 이 다음 작업의 input 으로 매끄럽게 흘러간다.
- Yule 은 이미 lifecycle / Obsidian / topic ledger 로 자산화의 절반은 했다. 부족한 절반은 **자산을 다시 꺼내 쓰는 흐름** — recall / context compression / 반복 작업 / 회고 자동화.

따라서 Hermes 도입의 진짜 가치는 코드 흡수가 아니라 **정책 흡수**. Yule 의 lifecycle 안에 "memory recall 시기 / 압축 트리거 / 회고 자동화 / scheduled work 안전 가드" 를 명시적으로 박는 게 본 phase 의 목표.

## 구현 위치 후보 (본 phase)

이번 phase 는 **정책 + 정책 보강 + 최소 helper** 까지로 한정. 코드 변경은 후속 phase.

- `policies/runtime/agents/engineering-agent/memory-policy.md` (신규)
- `policies/runtime/agents/engineering-agent/recall-policy.md` (신규)
- `policies/runtime/agents/engineering-agent/context-compression.md` (신규)
- `policies/runtime/agents/engineering-agent/self-improvement-flow.md` (신규)
- `policies/runtime/agents/engineering-agent/scheduled-automation.md` (신규)
- `agents/engineering-agent/agent.json` 의 `policies` 배열 갱신 (기존 정책 등록 패턴)

## 리스크와 다음 액션

- 리스크 1 — 정책 5 개 추가가 *#48 (팀 아키텍처 일반론)* 과 영역 겹침. 본 phase 정책은 모두 **memory / context / scheduling / improvement / recall** 에 한정 — 일반 아키텍처는 손대지 않음.
- 리스크 2 — *#25 (foundation 변경)* 과 영역 겹침. 본 phase 는 새 정책 추가 only — `lifecycle-mvp.md` / `obsidian-memory.md` / `role-profiles.md` 본문 수정 없음.
- 리스크 3 — 정책이 코드보다 앞서 가서 "정책만 있고 구현 없음" 상태 발생. 본 phase 정책은 모두 *향후 코드 후속 milestone* 명시 + Yule 이 이미 갖고 있는 부분(topic ledger, knowledge writer, supervisor) 에 anchor 를 둔다.

다음 액션:
1. decision note 작성
2. 5 개 정책 문서 작성 + agent.json 등록
3. 테스트 / 정책 self-check
4. progress comment + draft PR
