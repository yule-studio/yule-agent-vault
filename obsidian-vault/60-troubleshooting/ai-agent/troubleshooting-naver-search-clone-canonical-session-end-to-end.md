---
title: troubleshooting · canonical session 11917bf1e75d 종단 회복 시리즈
kind: troubleshooting
status: open
created_at: 2026-05-18
tags:
  - ai-agent
  - runtime
  - coding-executor
  - recovery
  - greenfield-bootstrap
related:
  - "[[troubleshooting-naver-search-clone-autonomous-intake-sandbox-block]]"
  - "[[engineering-agent-governance]]"
home_hub: "60-troubleshooting"
---

# troubleshooting — canonical session `11917bf1e75d` 종단 회복 시리즈

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-18 | tech-lead | naver-search-clone 풀스택 MVP canonical session 의 P0-T → P1-I 16 단계 디버깅 통합 기록. 각 단계마다 증상 / 원인 / 해결 / 회귀 가드 명시. |

## 한 줄 요약

`/engineer_intake` 로 들어온 외부 repo (`yule-studio/naver-search-clone`) full-stack 코딩 요청 한 건이 16 단계 의 개별 버그를 거치며 한 번도 동일 원인으로 멈추지 않고 매번 *다음* 단계로 전진했다. 각 단계의 root cause / fix / 자동 복구 경로를 다음 사고에서 같은 함정을 다시 밟지 않도록 기록.

## 공통 맥락

- **branch**: `fix/engineering-write-reply-routing`
- **canonical session_id**: `11917bf1e75d`
- **target repo**: `yule-studio/naver-search-clone`
- **issue anchor**: `#1`
- **모든 fix 원칙**:
  - 진단 단계는 "증상 → 가설 → 코드/DB 단서 → root cause" 로 명시
  - fix 는 코드 SSoT 한 곳 + 회귀 가드 (stdlib unittest) 동반
  - operator 가 새 intake 없이 같은 session 으로 자동 회복 가능해야 함
  - silent heal 금지 — warning log + audit field 노출

---

## P0-T. status surface visibility / work_order executor inventory 누락

### 증상
- `coding_execute queued=1` 상태로 멈춤. 그러나 inventory 에는 그 큐를 소비하는 executor 등록 없음.
- `eng-supervisor-watch` / `eng-discord-gateway` / member bot 들이 `yule runtime status` 에서 영원히 `UNKNOWN`.

### 원인
1. `runtime/services.py` 의 inventory 에 `eng-github-work-order-executor` 서비스 spec 자체가 없었다 — producer 는 enqueue 하지만 consumer 가 영원히 안 돈다.
2. supervisor / gateway / member bot 들이 자체 heartbeat 를 기록하지 않아 status surface 가 그들을 dead 로 본다.

### 해결
- `runtime/services.py` 에 `eng-github-work-order-executor` 추가, `ServiceKind.GITHUB_WORK_ORDER_EXECUTOR` 신규.
- `runtime/run_service.py` 에 `_run_github_work_order_executor` 분기 추가 (→ 이후 `runtime/work_order_executor_runner.py` 로 분리).
- supervisor / gateway / member bot 모두 자체 `heartbeat_loop` 등록 (`runtime/heartbeats.py`).

### 회귀 가드
- `tests/runtime/test_status_visibility_fixes.py`
- `tests/governance/test_code_audit.py::LiveKindToJobTypeWiringTests` — `JOB_TYPE_*` ↔ `_KIND_TO_JOB_TYPE` cross-check.

---

## P0-U. github_work_order `SKIPPED_NO_REPO` failed_retryable 적층

### 증상
- 승인 reply 후 `github_work_order queued=1` → 곧 `failed_retryable`.
- `result_json.error = github_work_order_no_repo`.

### 원인
- slash command intake (`_run_engineer_intake`) 가 `enqueue_github_work_approval` 을 호출할 때 `repo=None` 으로 호출.
- 그 결과 work_order payload 의 `repo_full_name` 이 빈 문자열 → executor 가 어디로 push 할지 모름.

### 해결
- `discord/commands/__init__.py` 에 `_extract_repo_from_session(session, prompt_text)` 추가 — `session.references_user` / `extra.coding_repo_full_name` / prompt 의 GitHub URL 순으로 canonical `owner/repo` 추출.
- executor 측 `_recover_repo_for_work_order` fallback 추가 — payload 가 비어도 session 에서 한 번 더 시도.

### 회귀 가드
- `tests/job_queue/test_work_order_repo_recovery.py`

---

## P0-V. work_order plan 누락 / SKIPPED_MISSING_PLAN 자동 복구

### 증상
- `result_json.error = github_work_order_missing_plan_or_issue` 로 stranded.
- producer 가 plan 을 빠뜨리고 enqueue 한 row 가 fix 후에도 영원히 깨어나지 않음.

### 원인
- `build_github_work_order_proposal` 의 P0-U fix 가 신규 흐름은 잡았지만, fix 이전에 enqueue 된 row 는 그대로 stranded.
- executor 가 plan 없으면 즉시 fail-fast 만 했음. payload 의 `repo_full_name` 만 있어도 minimal `RepoContract(fallback=True)` 로 plan 을 재구성할 수 있는데 그 self-heal 이 없었다.

### 해결
- `agents/job_queue/github_work_order_recovery.py` 신설:
  - `recover_plan_from_work_order(work_order)` — `_minimal_repo_contract` + `build_issue_auto_create_plan` 의 default body fallback 으로 plan dict 즉석 재구성.
  - `requeue_no_repo_failures` / `requeue_missing_plan_failures` — startup sweep.
- executor `process_job` Branch 2 가드를 강화: plan 누락 + repo 있음 → self-heal → 그대로 issue create 진행.
- LOC 분리: executor 1053 → 850, recovery 책임은 신규 모듈.

### 회귀 가드
- `tests/job_queue/test_work_order_repo_recovery.py::StartupRetryHookMissingPlanTests`

---

## P0-W. canonical session 의 qa-test 오분류 + coding_proposal 누락

### 증상
- `task_type = qa-test`, `executor_role = qa-engineer` — 풀스택 코딩 요청인데 QA 로 분류됨.
- continuation 이 `coding_dispatch_noop_reason = no_coding_proposal` 로 멈춤.

### 원인 (두 개 겹침)
1. `_KEYWORD_RULES` 의 QA_TEST 키워드에 bare `"qa"` 2-char substring → `naver-quasar` 류 무관 단어에 trip.
2. `stack_detector` 의 lexicon 이 영문 기반 (Next.js / NestJS / Postgres) 만 — 한국어 prompt (`프론트 / 백엔드 / 데이터베이스 / 도커`) 는 tier 0개로 평가 → keyword fallback 이 QA 의 약한 substring 매칭에 잡힘.
3. slash intake 가 `session.extra['coding_proposal']` 를 stamp 하지 않음 — 옛 engineering channel router 의 "코딩 권한 제안" 자유 phrase 경로에서만 stamp 됐다.

### 해결
- QA_TEST 키워드를 강한 phrase 만 인정: `regression test` / `회귀 테스트` / `qa engineer` / `qa-engineer` / `테스트 자동화` / `테스트 시나리오` / `테스트 케이스` / `테스트 커버리지` 등. bare `qa` / 단순 `회귀` 제거.
- `stack_detector` 에 한국어 tier alias 추가 (`프론트엔드` / `백엔드` / `데이터베이스` / `도커` / `회원가입` / `로그인` 등) + `풀스택` explicit hint 가 있고 tier 1개 이상 + `write_requested=True` 면 `FULL_STACK_APP` commit.
- `_ensure_coding_proposal_on_session` helper 추가 — slash intake 가 `recommend_authorization` + `_persist_coding_proposal` 호출 (idempotent).
- 신규 `repair_session_for_coding_dispatch(session_id, load_session_fn, update_session_fn)` — anchor 있는 stranded session 의 proposal 재구성 + task_type 재분류 + `promote_session_to_coding_ready` 재실행 + stale `write_blocked_reason` (qa-engineer 메시지) 정리.

### 회귀 가드
- `tests/engineering/test_session_recovery_qa_misclassification.py` (17 케이스)

---

## P0-X. continuation self-heal + startup sweep

### 증상
- `repair_session_for_coding_dispatch` 가 라이브러리 함수일 뿐 자동 호출되지 않음 → canonical session 그대로 멈춤.
- `coding_dispatch_noop_reason = no_coding_proposal` 가 매번 같은 자리에서 반복.

### 원인
- `promote_session_to_coding_ready` 가 proposal 없으면 `CONTINUATION_NOOP_NO_PROPOSAL` 반환. caller 가 따로 복구하지 않으면 영원히 멈춤.
- 이미 SAVED 로 마감된 row 는 worker 가 다시 picks 하지 않음 — startup sweep 가 필요.

### 해결
- `promote_session_to_coding_ready` 에 `session_prompt` / `auto_rebuild_proposal=True` 파라미터 추가 — proposal 누락 + prompt 있음 → 즉석 재구성 + audit (`rebuilt_by: continuation_self_heal`).
- `_continue_to_coding` 가 `session.prompt` / `session_id` 를 전달 → 다음 work_order 처리부터 자동 self-heal.
- 신규 `repair_stranded_coding_sessions(queue, ...)` startup sweep — SAVED + `coding_dispatch_queued=False` + `coding_dispatch_noop_reason=no_coding_proposal` row 자동 repair.

### 회귀 가드
- `tests/job_queue/test_continuation_self_heal.py` (9 케이스)

---

## P0-Y. coding_execute producer/consumer 분리 + marker semantics

### 증상
- `coding_execute queued=0` 인데 ready coding_job session 은 존재.
- 새 row 가 영원히 안 만들어짐.

### 원인 (chicken-and-egg deadlock)
- `dispatch_ready_coding_jobs(worker)` 호출이 `_build_process_job(spec).CODING_EXECUTOR` 분기 안의 `_process(job)` 안에서만 실행.
- 즉 `coding_execute` job 이 이미 있어야 producer 가 tick 한다 → 큐가 비면 영원히 안 돈다.
- 동시에 `coding_dispatch_queued` progress marker 가 coding_job=ready stamp 시점에 찍혀서 operator surface 가 "queued 됐다" 고 거짓말.

### 해결
- 신규 `runtime/coding_executor_runner.py` (180 LOC) — `run_coding_executor` 가 두 asyncio task 병렬:
  - `_producer_loop` (background): 매 10s `dispatch_ready_coding_jobs(worker)` 호출
  - `run_worker_loop` (consumer): 표준 drain
- `PROGRESS_CODING_JOB_READY` 신규 — proposal → CodingJob=ready stamp 시점.
- `PROGRESS_CODING_DISPATCH_QUEUED` 는 `_persist_dispatch_marker` (실제 queue row 생성 후) 가 stamp 하도록 의미 정정.

### 회귀 가드
- `tests/runtime/test_coding_executor_producer_decoupling.py` (7 케이스)

---

## P0-Z. cross-store invariant SSoT — phantom dispatch marker

### 증상
- `session.extra['coding_execute_dispatch'].job_id = "1779022229629-ffff03784aaf"` 있지만 그 id 의 queue row 가 통째로 없음.
- producer 가 marker 만 보고 "이미 dispatch 됨" 으로 영구 skip.

### 원인
- 옛 `ReadyCodingJob.has_been_dispatched()` 가 marker 존재만 보고 True 반환. queue 와의 invariant 검증이 없었다.

### 해결
- 신규 public SSoT `validate_coding_dispatch_marker(session, queue) -> DispatchMarkerCheck`:
  - state: `valid` / `missing` / `stale`
  - reason: `marker_job_id_not_in_queue` / `marker_wrong_job_type` / `marker_wrong_session_id` / `marker_row_terminal_or_inactive`
- `iter_ready_coding_jobs(..., queue=)` 가 queue-aware skip — phantom 이면 caller 가 re-enqueue.
- `dispatch_ready_coding_jobs(validate_marker_against_queue=True)` 기본 True. stale 발견 시 **warning 레벨** 로그 + `DispatchedCodingJob.stale_marker_reason` audit 채움.

### 회귀 가드
- `tests/job_queue/test_dispatch_marker_invariant.py` (14 케이스)

---

## P1-A. coding_execute long-running job 의 lease timeout reap

### 증상
- `coding_execute in_progress=1` 상태로 정상 실행 중이던 row 가 ~65s 후 `failed_retryable` 로 강제 전환.
- `result_json` 비어있음 → operator 가 timeout 인지 worker failure 인지 모름.

### 원인
- 기본 `pick_lease_seconds=60.0` 인데 coding pipeline (worktree + edits + tests + commit + push + PR) 은 자주 60s 초과.
- worker 측에 lease 갱신 API 없음 → supervisor 의 `reap_expired_leases` 가 active job 을 강제 회수.

### 해결
- 신규 `JobQueue.renew_lease(job_id, *, lease_seconds, worker_id=None, now=None)` — picked_until 만 갱신, state 그대로.
- `reap_expired_leases` 가 reap 한 row 에 `error="lease_expired"` + `reaped_at` + `previous_picked_by` + `previous_state` audit stamp.
- `runtime/coding_executor_runner.py` 에 background `_lease_keepalive_loop` task — 매 30s `queue.renew_lease(...)` 호출. 진짜 hang 이면 keepalive 가 멈춰서 reaper 가 정상 회수.
- 신규 `_recover_lease_expired_rows` startup hook — `lease_expired` reason 의 failed_retryable rows 자동 requeue.
- 신규 env: `YULE_CODING_EXECUTE_PICK_LEASE_SECONDS` (default 900s) / `YULE_CODING_EXECUTE_KEEPALIVE_INTERVAL_SECONDS` (default 30s).

### 회귀 가드
- `tests/runtime/test_coding_executor_lease_keepalive.py` (8 케이스)

---

## P1-B / P1-C. cross-repo worktree + branch reuse + dispatcher 비-overreach

### 증상
- 매 retry 마다 `edit_failed: _SubprocessError: subprocess failed: exit=255`.
- 동시에 producer 가 매 tick 새 fresh row 를 만들어 attempt 카운터 / backoff / max_attempts 무력화.

### 원인 (두 개 겹침)
1. `LocalGitWorktreeProvisioner` 가 orchestrator repo (`yule-studio-agent`) 를 그대로 target repo 로 쓰면서 `git worktree add -b` 가 (a) 잘못된 repo 에서 (b) 이미 존재하는 branch `agent/backend-engineer/issue-1-coding-execute` 와 충돌해 실패.
2. P0-Z phantom 분류가 `failed_retryable` / `failed_terminal` 도 phantom 으로 잘못 잡아 producer 가 매 tick fresh row 생성 → attempt 항상 0.

### 해결
- 신규 `_default_repo_root_resolver` — env JSON map / env search paths / orchestrator sibling 검색.
- 신규 `TargetRepoUnavailableError` / `WorktreeProvisionError` (reason 토큰 포함).
- `LocalGitWorktreeProvisioner.provision` idempotent — `-b` 가 "already exists" 면 자동으로 `-b` 없이 reuse.
- `process_job` 의 worktree 분기를 별도 try/except → `REASON_TARGET_REPO_MISSING` (terminal) / `REASON_WORKTREE_FAILED:<sub>` (non-terminal).
- `MARKER_STATE_PENDING_RETRY` (failed_retryable) + `MARKER_STATE_TERMINAL` (saved/failed_terminal) 분리 — producer 가 둘 다 skip. queue retry semantics 가 책임.
- task-log obsidian render 지원 (`NOTE_KIND_TASK_LOG` 를 `_DEFAULT_RENDER_KINDS` + `_M10B_AUTONOMOUS_KINDS` 등록).

### 회귀 가드
- `tests/job_queue/test_coding_executor_target_repo_and_retry.py` (11 케이스)

---

## P1-D. `target_repo_checkout_missing` 자동 복구

### 증상
- canonical session 의 latest job (`1779027211596-9215f2cf937b`) 이 `result.reason = target_repo_checkout_missing` 으로 `failed_terminal`.
- operator 가 사후에 checkout 을 만들거나 `YULE_CODING_EXECUTOR_REPO_ROOTS_JSON` 을 설정해도 row 가 그대로 stranded.

### 원인
- 옛 분류는 `terminal=True` 였다. repo checkout 부재는 *환경* 문제이지 *비즈니스* 실패가 아닌데도.
- recovery hook 자체가 없음.

### 해결
- worker 에서 `target_repo_unavail` 분기를 `terminal=False` 로 변경 (queue retry / backoff / max_attempts 가 자연 정지).
- 신규 `agents/job_queue/coding_execute_recovery.py`:
  - `recover_target_repo_missing_rows(queue, *, repo_resolver, ...)` — failed_retryable / failed_terminal 행을 sweep, resolver 가 디렉터리 확인하면 revive.
  - failed_retryable → `requeue_retryable`; failed_terminal → 직접 SQL `_revive_failed_terminal_row` (state machine 의 terminal end-state 제약을 명시적으로 우회).
  - `result_json['revivals'][]` 에 audit 기록.
- `runtime/coding_executor_runner.py` startup + periodic tick (60s) wiring.

### 회귀 가드
- `tests/job_queue/test_coding_execute_target_repo_recovery.py` (8 케이스)

---

## P1-E. stack-aware test command 선택

### 증상
- target repo (`naver-search-clone`) 가 Next/Nest/Postgres 스택인데 worker 가 `python3 -m unittest discover` 시도 → `test_failed`.

### 원인
- `SubprocessTestRunner` 가 `DEFAULT_TEST_COMMAND` 하나로 모든 repo 처리. `_resolve_test_command` 는 metadata override 만 봤다.

### 해결
- 신규 `agents/job_queue/coding_execute_test_command.py`:
  - `select_test_command(worktree_path, request_metadata, fallback_command) -> TestCommandSelection`
  - 우선순위: metadata override → JS/TS (`package.json` + `scripts.test` → `<pm> run test`; 없으면 pm-default) → Python (`pytest.ini` / `pyproject [tool.pytest]` / `manage.py`) → unittest discover fallback
  - Package manager 추론 (pnpm > yarn > bun > npm by lock files)
- `SubprocessTestRunner.run` 가 `test_summary['selection']` 에 strategy / command / package_manager 등 stamp.

### 회귀 가드
- `tests/job_queue/test_coding_execute_stack_and_materialization.py` (7 케이스 in StackAware section)

---

## P1-F. governed target repo auto-materializer

### 증상
- operator 가 수동으로 `naver-search-clone` 을 clone 해야만 작업 진행. production 운영 시 병목.

### 원인
- `_default_repo_root_resolver` 가 "이미 있는 checkout 찾기" 까지만 함. 다음 단계 (clone / fetch) 가 구현되지 않음.

### 해결
- 신규 `agents/job_queue/coding_execute_repo_materializer.py`:
  - `materialize_repo(*, repo_full_name, ...)` — opt-in (`YULE_CODING_EXECUTOR_REPO_AUTO_CLONE=1`) + owner allowlist (`YULE_CODING_EXECUTOR_ALLOWED_REPO_OWNERS=yule-studio`) + cache root (`YULE_CODING_EXECUTOR_REPO_CACHE_ROOT=~/.cache/yule/repos`).
  - 결과 토큰 7 종: `cloned` / `fetched` / `reused` / `refused_disabled` / `refused_owner` / `refused_invalid_repo_name` / `failed`.
  - idempotent: cache 있으면 `git fetch --prune` (실패 시 reuse), 없으면 `git clone --filter=blob:none`.
- `LocalGitWorktreeProvisioner.resolve_repo_root_for_request` 가 existing checkout 못 찾으면 → materializer → 그래도 실패면 `TargetRepoUnavailableError(materialization 정보 포함)`.

### 회귀 가드
- `tests/job_queue/test_coding_execute_stack_and_materialization.py` (RepoMaterializerTests 6 + ProvisionerResolutionChainTests 2)

---

## P1-G. greenfield / no-stack repo 는 bootstrap_required 로 surface

### 증상
- `naver-search-clone` 이 `.git` + README 만 있는 greenfield 인데 `SubprocessTestRunner` 가 silently `python3 -m unittest discover` 로 fallback → `ImportError: Start directory is not importable: 'tests'` → misleading `test_failed`.

### 원인
- `select_test_command` 가 시그널 0 일 때 unittest discover 로 fallback. greenfield / no-stack 케이스를 별도 reason 으로 surface 하지 않음.

### 해결
- 신규 strategy `bootstrap_required` + sub-reasons `empty_or_greenfield_repo` / `no_stack_detected`.
- Python signal 탐지 확장 (setup.py, requirements.txt, Pipfile, poetry.lock, tox.ini, setup.cfg, tests/*.py, top-level *.py 등).
- `_looks_greenfield` — `.git` / README / LICENSE / .editorconfig 만 있으면 True.
- `SubprocessTestRunner.run` 가 bootstrap_required 면 subprocess 호출 자체 안 함.
- worker 에 신규 `REASON_BOOTSTRAP_REQUIRED` — `terminal=True` (infinite retry churn 차단).
- RecordOnlyCodeEditor 감지 시 sub_reason 에 `+editor_record_only_insufficient` append.

### 회귀 가드
- `tests/job_queue/test_coding_execute_bootstrap_required.py` (9 케이스)

---

## P1-H. GreenfieldBootstrapEditor — deterministic scaffold execution

### 증상
- bootstrap_required 가 정확히 보이지만 실제로 scaffold 를 만드는 capability 없음. record-only editor 만 존재.

### 원인
- ordinary "edit existing code" path 외에 *bootstrap mode* 가 없었다. record-only editor 는 plan 노트만 작성.

### 해결
- 신규 `agents/coding/greenfield_bootstrap.py`:
  - `detect_bootstrap_mode(request, worktree_path) -> Optional[str]` — `MODE_GREENFIELD_FULL_STACK` / `MODE_GREENFIELD_PYTHON` / None
  - `plan_greenfield_scaffold(mode, request) -> BootstrapPlan` — deterministic minimal scaffold (full-stack 은 10 파일: `package.json` monorepo + `pnpm-workspace.yaml` + `docker-compose.yml` web/api/postgres + `.env.example` placeholder + `apps/web/{package.json, pages/index.tsx}` + `apps/api/{package.json, src/main.ts}` + `GREENFIELD_BOOTSTRAP.md`)
  - `apply_bootstrap_plan(*, worktree_path, plan, write_scope) -> BootstrapApplyResult` — write_scope governance + idempotent (never overwrite)
- 신규 `GreenfieldBootstrapEditor` (env-gated `YULE_CODING_EXECUTOR_GREENFIELD_BOOTSTRAP_ENABLED=1`):
  - non-greenfield → record-only delegation
  - greenfield + env off → `BootstrapLiveEditorUnavailable` 예외
  - greenfield + env on → scaffold 적용 + audit
- `build_live_executor` 의 `code_editor` slot 을 새 editor 로 교체.

### 회귀 가드
- `tests/job_queue/test_greenfield_bootstrap.py` (12 케이스)

---

## P1-I. `bootstrap_required:*` 자동 회복

### 증상
- canonical session 의 3 failed_terminal row (`1779058744308-…`, `1779058734281-…`, `1779058012948-…`) 가 operator 가 env opt-in 한 뒤에도 자동 재시도되지 않음.

### 원인
- recovery hook 이 `lease_expired` / `target_repo_checkout_missing` 만 다룸. `bootstrap_required:*` 는 빠져 있었다.

### 해결
- `coding_execute_recovery.py` 에 `recover_bootstrap_required_rows(queue, *, repo_resolver, ...)` 추가:
  - `result.reason` startswith `bootstrap_required:` + sub-token classifier 가 *capability-driven* (`editor_record_only_insufficient` / `live_editor_unavailable`) 만 recoverable 로 인정.
  - `scaffold_apply_failed:*` 는 operator intervention 필요 → revive 안 함.
  - Two-gate: env opt-in AND repo checkout 실재 → revive.
  - failed_retryable → `requeue_retryable`; failed_terminal → 직접 SQL revive + `revivals[]` audit.
- runtime startup + periodic tick (60s) wiring.

### 회귀 가드
- `tests/job_queue/test_bootstrap_required_recovery.py` (13 케이스)

---

## 운영 가이드 — canonical session 같은 케이스 재발 시

1. **상태 확인 순서**:
   1. `yule runtime status` — 어느 큐에서 멈췄나
   2. session.extra (`yule engineer show --session <id>`) — `coding_execute_progress` / `coding_blocked` marker 의 `status` / `selection` / `sub_reason`
   3. `result_json.reason` — `bootstrap_required:<sub>` / `target_repo_checkout_missing` / `lease_expired` / `test_failed` 등 분류
   4. `result_json.revivals[]` — 이미 한 번 revive 됐는지

2. **자주 만나는 reason 별 조치**:
   - `target_repo_checkout_missing` → operator 가 sibling checkout 또는 env JSON map. 또는 `YULE_CODING_EXECUTOR_REPO_AUTO_CLONE=1` opt-in.
   - `bootstrap_required:editor_record_only_insufficient` → operator 가 `YULE_CODING_EXECUTOR_GREENFIELD_BOOTSTRAP_ENABLED=1` opt-in.
   - `bootstrap_required:scaffold_apply_failed:*` → write_scope / 권한 점검. 자동 revive 안 됨.
   - `lease_expired` → 다음 runtime restart 에 자동 requeue.

3. **운영자가 외울 명령 1줄**:
   ```bash
   yule run-service eng-coding-executor   # restart 한 번이면 startup sweep 가 모든 recoverable 분류 자동 처리
   ```

## 다음에 같은 함정 피하기

- **모든 fail 분기에 specific reason 토큰** — generic `test_failed` / `edit_failed` 하나에 다른 의미 합치지 말 것.
- **모든 terminal 분류는 의도된 결과** — 환경 의존 fail 은 `terminal=False` + 별도 recovery hook 으로.
- **모든 self-heal 은 audit + warning log** — silent heal 금지. operator 가 "왜 갑자기 다시 queued 됐는지" 알 수 있어야 한다.
- **모든 cross-store invariant 는 SSoT 함수** — `session.extra` vs `job_queue` 같은 두 store 의 진실은 한 곳에서 판정.
- **모든 producer/consumer 는 분리된 task** — chicken-and-egg deadlock 방지.

## 참고

- 모든 fix 의 commit 기록은 branch `fix/engineering-write-reply-routing` 의 P0-T → P1-I 시리즈.
- 테스트는 모두 stdlib unittest (`python3 -m unittest discover -s tests -t .`) 로 CI baseline 충족.
- governance/code_audit hard rail (split_now 0) 유지.
