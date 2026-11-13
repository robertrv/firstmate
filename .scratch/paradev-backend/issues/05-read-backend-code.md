# 05 Read Firstmate's backend seams

Status: resolved
Type: research

Read `bin/fm-backend.sh` (dispatcher), `bin/backends/orca.sh` + `docs/orca-backend.md` (app-owned worktree, closest template for Option B), `bin/backends/zellij.sh` + `docs/zellij-backend.md` (explicit-only, hash-poll fallback), `docs/architecture.md`, `docs/configuration.md` "Runtime backend", and `.agents/skills/firstmate-orca`.

## Acceptance

A short note appended under `## Comments` listing: every function a backend must implement, how a backend is registered as explicit-only, where metadata keys are declared, which tests every backend must have (`tests/fm-backend-orca.test.sh`, `tests/fm-backend.test.sh`, `tests/fm-teardown-endpoint-safety.test.sh`, ...), and how `docs/verification/runtime-backends.md` is expected to be filled.

## Comments

### 2026-11-13 - backend seams read (agent)

Read `bin/fm-backend.sh`, `bin/backends/orca.sh` + `docs/orca-backend.md`, `docs/zellij-backend.md`, `docs/architecture.md` "Runtime session backends", `docs/configuration.md` "Runtime backend", `.agents/skills/firstmate-orca`, plus the orca call sites in `fm-spawn.sh`, `fm-teardown.sh`, `fm-watch.sh`, `fm-control-lib.sh`, `fm-test-run.sh`. Line numbers are as of b6d65c74.

**There is no plugin interface.** `fm-backend.sh` dispatches with a hard-coded `case "$backend"` per primitive, and other scripts hold their own per-backend `if [ "$BACKEND" = orca ]` branches. Adding `paradev` means editing every table below, not just dropping in `bin/backends/paradev.sh`.

#### 1. Functions an adapter implements (`bin/backends/paradev.sh`, prefix `fm_backend_paradev_`)

Required by every dispatch arm (orca implements all of these; it is the template):

| Function | Dispatcher | Contract notes |
|---|---|---|
| `_capture <target> <lines> [label]` | `fm_backend_capture` | bounded tail; also used by orca as `target_exists` (capture 1 line) |
| `_send_key <target> <key> [label]` | `fm_backend_send_key` | shared key names Enter, Escape, C-c, C-u; refuse unsupported keys loudly (orca refuses Escape/C-u) |
| `_send_text_submit <target> <text> <retries> <enter-sleep> <settle> [label]` | `fm_backend_send_text_submit` | type once, retry Enter only; echo empty on confirmed delivery; confirmation must come from the composer classifier |
| `_composer_state <target> [label]` | `fm_backend_composer_state` | thin wrapper: capture + caps descriptor fed to `fm_composer_classify_screen` in `bin/fm-composer-lib.sh` (orca: `_composer_capture`, `_composer_caps`, `_json_text`) |
| `_kill <target>` | `fm_backend_kill` | already-gone = 0 silent; nonzero only when it can prove the close failed (orca: CLI missing = 1) |
| `_remove_worktree <worktree-id>` | `fm_backend_remove_worktree` | ONLY app-owned-worktree backends (today only orca; the `case` must gain `paradev`) |
| `_worktree_path <worktree-id>` | `fm_backend_worktree_path` | resolves recorded id to path; teardown compares it to `worktree=` before release |
| `_runtime_check` | `fm_backend_runtime_check` | early spawn check before any repo mutation (orca: `orca status --json` ready). For paradev: probe a real call, not `superset status` (spike finding 3) |
| `_tool_check` | (internal, called by runtime_check) | |

Spawn-time helpers called directly from `fm-spawn.sh`, not through the dispatcher (orca names): `_repo_ensure <project>`, `_worktree_create <project> <name>` (exit 2 + partial output means "created, then failed", triggers abort cleanup), `_terminal_create <worktree-id> <title>`, `_send_text_line`, `_send_literal`, `_launch_env_args` (only if launch-env allowlist via tmux session is supported). `fm-spawn.sh` also has its own `case "$BACKEND"` send helpers at ~5496/5522/5540 and `parse_orca_worktree_result` / `spawn_orca_reclaim_launch_file` (~1747, ~1769).

Optional, default to "not supported" when absent:
- `_busy_state` (`fm_backend_busy_state`, default prints `unknown`). Paradev has native `agents explain` state, so this is the natural place for it once issue 03 is resolved; only the exact word `busy` is consumed.
- `_agent_state` (`fm_backend_agent_state`, default `unverified`; states alive/dead/missing/ambiguous/unreadable). Not needed for v1 (no secondmates); orca does not implement it.
- `_visible_capture` + entry in `FM_BACKEND_VISIBLE_CAPTURE` (orca is absent: history read only; `terminals read --max-lines` is likely the same).
- Push events: `_events_capable`, `_wait_transition`, `_commit_transition`, `_clear_transition` + `fm_backend_has_push` (herdr/rex/t3code only). Skip for v1; watcher polls.
- `_validate_harness` (only t3code restricts harnesses).

#### 2. Registration and explicit-only

Every place to edit in `bin/fm-backend.sh`:
- `FM_BACKEND_KNOWN` and `FM_BACKEND_SPAWN` (~l.100) - add `paradev`.
- `fm_backend_source` (~l.1028): one `case` arm listing sibling libs (orca: `fm-composer-lib.sh`; zellij/paseo also `fm-backend-hometag-lib.sh`) and one `_FM_BACKEND_PARADEV_SOURCED` guard arm.
- Every dispatcher `case` listed above, plus `fm_backend_target_exists` (~l.1448).
- `fm_backend_required_tools` (~l.540): orca is `'orca'` (no treehouse); paradev would be `'superset jq'`. Add a `fm_backend_required_tool_available` arm if the binary needs custom resolution.
- `fm_backend_target_of_meta` (~l.603): orca returns `terminal=` instead of `window=`.
- `fm_backend_validate_task_endpoint` (~l.703, orca arm ~l.807): metadata-only cleanup identity check; requires exact `endpoint_task_id`, `window=fm-<id>`, atom-valid terminal, shape-valid worktree id. Needs a paradev arm (and a `fm_backend_paradev_*_valid` shape check if the workspace id is not a plain atom; UUIDs pass `fm_backend_endpoint_atom_valid`).
- Header comment and the "Verified backend adapters" comment block.

**Explicit-only is by omission**: a backend is explicit-only simply by having no branch in `fm_backend_detect` (~l.221). Convention is a comment there saying why it is deliberately not detected (see the Paseo/Tern comments), plus a test `test_backend_<name>_is_never_autodetected` in `tests/fm-backend.test.sh` (~l.491) proving that ambient markers (here `SUPERSET_WORKSPACE_ID`, any Paradev bundle id) never select it while `FM_BACKEND=paradev` and `config/backend` do. Also add `paradev` to the "never auto-detected" sentence in `docs/configuration.md` "Backend selection order".

Other per-backend tables outside the dispatcher (paradev must be added or deliberately left out):
- `bin/fm-spawn.sh`: secondmate refusal (~2396), endpoint creation `case` (~5381), `META_WINDOW=$W` (~7310), meta write block (~7397), relaunch-preserved key list (~7329), abort cleanup (`ORCA_ABORT_CLEANUP`, ~1827, ~5848), Treehouse skip (~6104), project capacity (~4262).
- `bin/fm-teardown.sh`: `require_orca_worktree_id`/`require_orca_terminal` (~1546), `require_orca_worktree_path_match` (~2625), the ship path-match gate before release (~3805), release (~4068), the generic-close exclusion (~4138), lock backend (~591), cleanup recovery (~729, ~1885, ~4034), secondmate child paths (~3286-3616).
- `bin/fm-watch.sh` ~806 (target = `terminal=`), `bin/fm-control-lib.sh` `fm_control_backend_supports_key` (~342), `bin/fm-bootstrap.sh` install hint (~872) and treehouse-not-needed checks (~1411, ~1500, ~1541), `bin/fm-supervise-daemon.sh` (refuses unsupported supervisor backends; nothing to add), `bin/fm-claude-trust.sh` (~107: Claude trust refuses a path that is not a linked worktree sharing the project's git common dir - Paradev worktrees are real `git worktree`s, so should pass, verify).

#### 3. Where metadata keys are declared

There is no schema file. Keys are declared in three places that must agree:
1. Written by `bin/fm-spawn.sh` meta block (~l.7340-7420), one `if [ "$BACKEND" = ... ]` per backend.
2. Preserved across relaunch by the `split("window endpoint_task_id ... orca_worktree_id terminal ...")` list in `preserve_relaunch_meta` (~l.7329). Every new key must be added here or relaunch duplicates it.
3. Documented in `docs/configuration.md` "Task metadata" (one bullet per backend) and the backend doc's "Task shape and metadata" block.
Read via `fm_meta_get` (last value wins) and validated via `fm_backend_meta_exact_value` (exactly one line). Orca shape: `backend=orca`, `window=fm-<id>`, `terminal=<handle>`, `orca_worktree_id=<repo id>::<abs path>`, `worktree=<abs path>`, `endpoint_task_id=<id>`. Paradev proposal: `backend=paradev`, `window=fm-<id>`, `terminal=<sessionId>`, `paradev_workspace_id=<ws>`, `paradev_project_id=<id>`, `worktree=<path>` (orca reuses the generic `terminal=` key; reusing it means `fm_backend_target_of_meta` and `fm-watch.sh` only need the backend name added).

#### 4. Tests every backend has

- `tests/fm-backend-<name>.test.sh` - the portable fake-CLI suite (orca: `make_orca_fakebin` + ~55 cases): capture/compose/send/key/kill/remove/path primitives, JSON error handling, runtime check, spawn writes metadata and launches, spawn refuses secondmate / not-ready / non-isolated worktree, spawn cleanup on each failure point, peek/send/crew-state route through meta, scout and ship teardown including id-path mismatch, missing id, missing terminal, missing path, error JSON, `test_dispatcher_sources_<name>_and_routes_primitives`. Copy orca's file as the starting point.
- `tests/fm-backend-<name>-smoke.test.sh` - optional real-binary smoke (zellij, cmux, paseo, tern, hc; orca has none). Gated, skipped when the binary is absent; the spec's "spawn two, peek, send, teardown" belongs here.
- `tests/fm-spawn-<name>-worktree.test.sh` - only for app-owned worktrees (orca).
- `tests/fm-backend.test.sh` - add the explicit-only test, a `test_backend_validate_spawn_accepts_<name>` case (~1069), and a dispatch-tables case like `test_backend_dispatch_tables_include_tern` (~875: required tools, visible capture, no push).
- `tests/fm-teardown-endpoint-safety.test.sh` - add a valid-record fixture (~367) and the backend to the empty-target loop `for backend in tmux herdr zellij orca ...` (~425); orca also has its own close-failure-under-force case (~1513).
- `tests/fm-bootstrap.test.sh` - required-tools delta (orca drops treehouse).
- Test runner registration in `bin/fm-test-run.sh`: family mapping for the new test files (~440, else they are `unclassified` and refused under `--jobs`, see `tests/fm-test-run.test.sh` ~1433), `expected_gate_skip_for_family` (~478, `optional-binary`), `list_known_families` (~498), path-to-family mapping for `bin/backends/paradev*` (~1586), and a timing entry (~747). `bin/fm-test-isolation-proof.sh` (~152) keeps a new backend test serial until proven isolated.
- No CI workflow lane exists for orca/zellij; "Experimental; no dedicated real-backend CI lane" is the expected status.

#### 5. How `docs/verification/runtime-backends.md` is filled

It is maintainer-audience, version-scoped evidence, not a guide. A backend section (`## Paradev`) contains: the exact binary and app version and OS verified against (orca: `/usr/local/bin/orca`, Orca.app 1.4.116); the real commands run in a `sh` block and the observed output fields in a `text` block; which response fields are accepted and which speculative shapes are deliberately rejected; the regression commands (`tests/fm-backend-<name>.test.sh`, `tests/fm-backend.test.sh`, `tests/fm-bootstrap.test.sh`) with one sentence on what the fake suite covers. Zellij uses a guarantee / command shape / result table instead. Rules from the file's preamble: no task chronology, branch names, temp homes, local paths, process ids or thread ids (those stay in private reports or PR evidence); date every claim. Also add a row to the "Endpoint close" table (~l.351: what `_kill` returns for already-gone vs failed close) and the backend to the "Cleanup endpoint identity" test list (~l.320). `docs/<name>-backend.md` links to `verification/runtime-backends.md#<name>`, and `docs/documentation-audiences.json` needs the new doc registered (path with audience `operator-current` ~l.449, and a source->target link to the verification file ~l.99); README.md docs list ~l.230 too.

#### Takeaways for issue 06

- Option B maps almost 1:1 onto orca: `workspace_create` ~ `orca worktree create`, `agents create` ~ `terminal create`, `terminal=` = sessionId, `workspaces get --json | jq .worktreePath` ~ `_worktree_path`, `workspaces archive` ~ `_remove_worktree` (only after the teardown path-match gate, which already exists for orca).
- Orca reads busy state from harness lifecycle hooks only; paradev can ship v1 the same way and add native `_busy_state` later (issue 03).
- A `.agents/skills/firstmate-paradev` operator skill mirroring `firstmate-orca` is the precedent, registered in `agent-skill-trigger-index`.
