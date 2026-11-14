# 08 Tests: unit and smoke

Status: resolved
Type: task
Blocked by: 06

Copy the pattern of `tests/fm-backend-orca.test.sh` (stubbed `superset` on PATH) and add a live smoke test: spawn two crewmates, peek, send, teardown. Wire them into the CI lanes the way other backends are wired.

## Comments

### 2026-11-14 - live smoke test added and wired (agent)

**Unit half.** Already delivered by ticket 06: `tests/fm-backend-paradev.test.sh` (keyed fake `superset` on `PATH`, the Orca pattern) plus the Paradev cases in `fm-backend.test.sh`, `fm-teardown-endpoint-safety.test.sh`, `fm-bootstrap.test.sh`, `fm-busy-state.test.sh` and `fm-test-run.test.sh`. Nothing to copy again.

**Smoke half: `tests/fm-backend-paradev-smoke.test.sh`.** Talks to the real Paradev host through the dispatcher and the adapter's spawn primitives (the calls `fm-spawn.sh` and `fm-teardown.sh` make). One run, 16 checks, about 28 s:
- preflight: `runtime_check`, and `project_id` resolves the clone path back to the given id;
- spawn two crewmates (workspace with its own branch and worktree, plus one plain shell terminal each); the first send goes out right after `terminals create`, as `fm-spawn.sh` does;
- each worktree is its own git root on its `spike/` branch, distinct from the clone and from the other one; `worktree_path` reads it back by id; both workspaces are listed under the `fm-spike` tag;
- a second create on a live crewmate's branch refuses (adapter `alreadyExists` path) and leaves that crewmate intact;
- task metadata built from the live ids validates through `fm_backend_validate_task_endpoint`; a cross-workspace `terminal=` refuses;
- peek and send: each shell starts in its own worktree (`pwd`); `send_literal` does not submit and `send_key Enter` does; no cross-talk between the two screens; dash-leading and `=` text arrive literally; `send_text_submit` runs the line; `C-c` interrupts a running `sleep`; `Escape` and `C-u` are accepted;
- `busy_state` is `unknown` for a shell; `target_exists` separates a live terminal from an unknown id;
- teardown in `fm-teardown.sh`'s order: `kill` (proved by the inventory re-read), repeat kill succeeds, worktree survives the close, recorded id still resolves to the recorded path, `remove_worktree` archives and the folder is gone, repeat archive succeeds (`notFound`); crewmate b keeps answering after a's teardown, then tears down the same way;
- final check: no smoke workspace left in `fm-spike`, no smoke branch left in the clone.

Commands use the shell, not an agent: no tokens are spent and nothing is ever sent to an agent, so the "never send while `starting`" rule cannot be hit. Marker lines use shell arithmetic (`echo m-$((40+2))` must print `m-42`), so a match proves the line ran, not just that it was typed.

**Safety and opt-in.** The test creates real workspaces in the captain's app and real branches in a project clone, so it only runs when `FM_PARADEV_SMOKE_PROJECT=<Paradev project id>` is set (otherwise `skip:`; it also skips without `superset`/`jq`/`git`, a host that answers `projects list` within 20 s, or that project on this host). Names are `fm-spike-smoke-<pid>-<rand>-{a,b}` on `spike/fm-smoke-<pid>-<rand>-{a,b}`, tag `fm-spike` (exported as `FM_PARADEV_TAG`). Cleanup is one guarded function run from an `EXIT` trap (`INT`/`TERM` become exits): it re-reads each workspace with `workspaces get` and archives it only when its name and branch are this run's; it also catches a workspace whose create reply was lost by exact run-unique name; then it deletes only this run's two branches. Nothing is archived by tag.
Cleanup proven live on three paths: an injected assertion failure right after both spawns, and two `SIGTERM`s while both crewmates were live (exit 143). Each time: 0 workspaces in `fm-spike`, 0 `spike/*` branches.
Run it: `FM_PARADEV_SMOKE_PROJECT=b9858b1a-100c-47b1-8304-a268c314eca6 bin/fm-test-run.sh --family paradev`.

**Wiring (`bin/fm-test-run.sh`).** The `paradev` family arm now holds the smoke test too (expected gate skip `optional-binary`, like the cmux/zellij/tern smoke tests). It joins the portable serial lane by derivation, so CI's macOS and Linux serial shards run it and record a gate skip there (no Paradev on CI runners). Serial weight hint `40` ms (the skip cost, like the other smoke tests). `tests/fm-test-run.test.sh` asserts the family selects it.

**Evidence for ticket 07 docs** (superset 2.0.8, macOS):
- Create-to-output for a send issued right after `terminals create`: 3 to 4 s across five runs, and the input was never lost. Only an observation: it does not prove a slow shell start can never drop it.
- A fresh `terminals create` shell starts in the workspace's worktree (the typed `cd` is belt and braces).
- Duplicate create on the same branch: Paradev replies `alreadyExists: true`; the adapter refuses with "a workspace for branch ... already exists" and archives nothing.
- `send_text_submit` on a plain shell returns composer verdict `unknown` (a shell screen has no composer), yet the line runs. Expected for a shell; a real harness composer is not covered here (no agent is launched).
- `C-c` interrupts a foreground command; `Escape` and `C-u` are accepted keys.
- `terminals read` returns the current screen, and the shell's earlier lines stay on it, so peek works for both crewmates. Whether Paradev can join `FM_BACKEND_VISIBLE_CAPTURE` (bin/fm-backend.sh) is a docs/backend decision outside my files: `terminals read` is the visible screen only, with no scrollback.
- Possible docs regression entry point: `tests/fm-backend-paradev-smoke.test.sh` with the opt-in variable above.

**Quality bar.** `bin/fm-lint.sh` clean on the changed files (explicit paths, full source following) and in default changed-file mode (ShellCheck 0.11.0, actionlint 1.7.12). `bin/fm-test-run.sh --family paradev`: 2/2 pass with the opt-in (live), and 1 pass + 1 gate skip without it. `tests/fm-test-run.test.sh` (18 min) and `tests/fm-backend.test.sh` pass through the runner. `bin/fm-test-run.sh --check-coverage`: ok (342 scripts, `serial_max_ms=1197625`, unchanged from the base).
Pre-existing, not fixed: `--check-coverage` fails ("portable serial shards share scripts") when the checkout itself lives under `$TMPDIR` (`/var/folders/...`); the base commit fails the same way there and passes from any other path (checked from the scratchpad and this worktree). No real workflow checks out under `$TMPDIR`, so I left it alone.
Not covered here: Paradev's native `working`/`idle` states with a real harness, and `fm-spawn.sh`/`fm-teardown.sh` end to end (both need a real agent and touch the project clone's remotes). That is ticket 09's dogfood.

`fm-spike` was left empty (`superset workspaces list --local --tag fm-spike` returns `[]`) and the project clone has no `spike/*` branches.
