# 06 Implement `bin/backends/paradev.sh`

Status: resolved
Type: task
Blocked by: 03, 05

Implement the adapter contract from `../spec.md` using the operation map in `../spike-findings.md`. Selected by `FM_BACKEND=paradev` or `config/backend`, explicit-only, never auto-detected, macOS only. Always pass `--json --local` and parse with `jq`.

## Acceptance

- spawn / send / peek / state / list-live / teardown / preflight implemented.
- Teardown refuses unless Firstmate's landing checks pass, and verifies the recorded workspace id and worktree path before archive.
- Tag every created workspace `firstmate`; list-live and recovery use only recorded ids, never the tag.
- Metadata: `backend=paradev`, `workspace=`, `terminal=`, `worktree=`, `paradev_project=`.

## Comments

### 2026-11-14 - backend implemented (agent)

**What landed** (test-first; every seam from ticket 05):

- `bin/backends/paradev.sh`: preflight (`superset` + `jq`, then a real `projects list --local --json` probe, never `superset status`), project lookup by exact path, workspace create, terminal create, send line / literal / key, capture, composer state, verified submit, kill, archive, worktree path, native busy state, target-ready.
- New seam in `bin/fm-backend.sh`: `fm_backend_owns_worktree` (`orca paradev`), `fm_backend_worktree_id_key`, `fm_backend_worktree_owner_label`. The Orca-only branches in `fm-spawn.sh`, `fm-teardown.sh`, `fm-watch.sh`, and `fm-bootstrap.sh` now ask this seam instead of `= orca`, so Paradev reuses Orca's spawn abort cleanup, recovery record, path-match gate, and teardown order rather than copying them. Orca keys and messages are unchanged (its suite passes untouched).
- Dispatcher: known/spawn lists, `superset jq` required tools (no tmux, no treehouse), source arm, every primitive arm, metadata-only endpoint validation, explicit-only comment in `fm_backend_detect`.
- `fm-control-lib.sh`: Escape, Enter, C-c, C-u supported. `fm-busy-lib.sh`: native `working` is trusted as busy only (idle stays unknown), like tern. Bootstrap: `MISSING_MANUAL: superset` hint. Test runner: `paradev` family, timing hint, path map, isolation-proof serial note.
- Tests: `tests/fm-backend-paradev.test.sh` (46 cases, keyed fake `superset`), plus cases in `fm-backend.test.sh` (never auto-detected, with a mutation check that it goes red if detection is added; dispatch tables; selector), `fm-teardown-endpoint-safety.test.sh` (valid/inconsistent records, close failure refuses under `--force`), `fm-bootstrap.test.sh`, `fm-busy-state.test.sh`, `fm-test-run.test.sh`.

**Lifecycle as built.**
Spawn: resolve the project id from the clone path, `workspaces create --local --project P --name fm-<id> --branch <ship branch or fm/<id>> --skip-branch-prefix [--base-branch B] --tag firstmate --json`, refuse `alreadyExists: true` without archiving it, read `worktreePath` back with `workspaces get`, run the isolated-worktree assertion, `terminals create` (plain shell), then Firstmate's normal typed `cd` + launch. Firstmate launches the harness itself (not `agents create`), so every harness and Firstmate's hooks work; Paradev's native state still arrives for Claude (checked live: a Claude typed into the shell went `working` then `idle`).
Teardown: Firstmate's landing checks, then `workspaces get` must resolve the recorded id to the inspected path, then branch delete, `terminals close`, an inventory re-read that proves it gone, then `workspaces archive <id>` (result must list the id as deleted or notFound, with no failures).
List-live and recovery are metadata-driven (the watcher walks task records and probes `terminals list` for the recorded terminal). Nothing lists or acts on workspaces by tag; the tests assert `workspaces list` is never called.

**Metadata (deviation from the acceptance line, on purpose):** `backend=paradev`, `window=fm-<id>`, `endpoint_task_id=<id>`, `terminal=<workspace-id>:<terminal-id>`, `paradev_workspace_id=<id>`, `paradev_project_id=<id>`, `worktree=<path>`.
Bare `workspace=` / `paradev_project=` would break the repo convention that backend-specific keys carry the backend prefix (`orca_worktree_id`, `cmux_workspace_id`, ...) and was ticket 05's proposal too. `terminal=` carries both ids because every terminal call needs both; validation refuses a `terminal=` whose workspace half differs from `paradev_workspace_id`.

**Live checks** (throwaway `fm-spike` workspaces `fm-spike-06a/b/c`, branches `spike/06a/b/c`, all archived and branches deleted; `workspaces list --tag fm-spike` is empty): every adapter primitive against superset 2.0.8. Facts found that the spike had not recorded:
- `workspaces create` reply has `tags: null` and no `worktreePath`; `workspaces get` also shows `tags: null` (only `list` shows tags). Path must be read back by id.
- `terminals close` and `workspaces archive` are idempotent (repeat calls report `disposed` / `deleted` again), so their replies prove nothing; kill re-reads `terminals list`.
- A plain shell terminal reports `agents explain` state `none`.
- The CLI rejects a separate `--text` value starting with `-` (`Option --text requires a value`); the adapter sends `--text=<value>` (checked live: `--dash`, `a=b=c`, quoted spaces arrive literally).
- This very workspace (the one I worked in) carries the `firstmate` tag, which confirms tag-based ownership would have been wrong.

**Quality bar.** `bin/fm-lint.sh` clean (changed-file mode, and explicit paths with full source following). Suites: see the result line below. Pre-existing failures on this machine, unrelated to the change: `tests/fm-spawn-orca-worktree.test.sh` needed `tmux` (`tmux is required for the Orca source locale regression`) and `tests/fm-bootstrap.test.sh` failed with `watcher failed during hygiene check (exit=127)` (see the follow-up result below for the real cause); The first two failed identically in a baseline run before any change. My new bootstrap case was run on its own and passes.
`tests/fm-backend-t3code.test.sh` needed Python 3.11+ `tomllib`; `python@3.12` was installed with Homebrew, after which its only failure is the live Codex case, because Paradev's `~/.paradev/bin/codex` wrapper exits 127 when no real Codex CLI is installed (with that directory off `PATH` the suite passes). Codex was left uninstalled on purpose.

**Notes for the next tickets.**
- 07 docs: `docs/paradev-backend.md`; `docs/configuration.md` runtime-backend list, the "never auto-detected" sentence, and a Task metadata bullet with the keys above; `docs/verification/runtime-backends.md` `## Paradev` section (superset 2.0.8 facts above, an "Endpoint close" row: Paradev re-reads `terminals list` and refuses while listed or unreadable), the cleanup-identity test list; register the doc in `docs/documentation-audiences.json` and the README docs list. Setup the doc must state: the clone Firstmate spawns from must itself be a Paradev project at that exact path (`superset projects create --import <path> --local`; spawn refuses otherwise and never registers one), Paradev's Claude hooks must load in the config dir Claude reads (issue 03), macOS only. Limits: no secondmates; no `fm-control exit/relaunch` (no recovery-grade `agent_state`, same as Orca); no viewport capture or push events; launch-env allowlist names resolve from the Paradev terminal's environment (spawn warns). Decide whether a `firstmate-paradev` skill is warranted (Orca has one).
- 08 smoke: add `tests/fm-backend-paradev-smoke.test.sh` to the `paradev` family arm in `bin/fm-test-run.sh` (`families_for_test`), gate on `superset` + a working `projects list`. `FM_PARADEV_TAG=fm-spike` puts its workspaces in the spike folder; use `spike/` branches. `terminals read` returns the visible screen only, so after a live check Paradev can join `FM_BACKEND_VISIBLE_CAPTURE`. Worth timing a send issued immediately after `terminals create` against a slow shell start (my runs, about 0 to 1 s after create, were fine).
- 09 dogfood: the Captain workspace may keep the `firstmate` tag; it is never treated as a crewmate.
- Follow-up, shared with Orca and not changed here: when a spawn's terminal create fails and the abort's own archive also fails, the recovery record has no `terminal=`, and `fm_backend_validate_task_endpoint` refuses it ("missing terminal"), so `fm-teardown.sh` cannot clean it without a hand edit. Relaxing that check for `cleanup_recovery=` records is a safety decision for a separate ticket.
- Follow-up: `fm_backend_agent_state` for Paradev (so `fm-control exit/relaunch` work) needs a process-level proof; `agents explain` cannot give one (`none` means both "plain shell" and "unknown terminal id").

**Result (final run, 2026-11-14).** `bin/fm-test-run.sh` over the paradev, orca, backend, teardown-endpoint-safety, busy-state, bootstrap, control, teardown, backlog-atomicity, zellij, hc, watch-triage and spawn-orca-worktree suites: all green except the two tmux-only failures above. `tests/fm-test-run.test.sh` passed separately.
That run also exposed a pre-existing macOS failure in `tests/fm-control.test.sh` (`fm-hermes-lib.sh: line 25: args[@]: unbound variable`; reproduced on a clean checkout of the base commit): bash 3.2 treats an empty array as unbound under `set -u`. Fixed in `bin/fm-hermes-lib.sh` with the `${args[@]+"${args[@]}"}` form; `tests/fm-control.test.sh` now passes 49/49.
Reported, not fixed: `tests/fm-hermes-harness.test.sh` fails on macOS at its NUL-argv assertion because `fm_hermes_process_matches` reads `/proc/<pid>/cmdline`, which only Linux has. Whether macOS needs its own argv source is a design question for a separate ticket.

**Follow-up run (2026-11-14, tmux 3.8 and python@3.12 installed).** `tests/fm-spawn-orca-worktree.test.sh` now passes.
`tests/fm-bootstrap.test.sh` still failed with exit 127: the cause was never tmux but GNU `timeout`, which macOS does not ship; three record-hygiene cases called bare `timeout`. Fixed by sourcing `bin/fm-timeout-lib.sh` and using its portable `fm_run_timed`; the suite now passes 42/42 with no skips.
`tests/fm-backend-t3code.test.sh`: the Python 3.11+ cases now run and pass; the only failure is `installed Codex failed the tracked configuration guard`, because `codex` on `PATH` is Paradev's `~/.paradev/bin/codex` wrapper, not the Codex CLI. With that directory removed from `PATH` for the run only, the suite passes (88 ok, `skip: live: codex absent`).
