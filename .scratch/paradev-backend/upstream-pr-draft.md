# Draft: upstream PR for the Paradev backend

Branch: `paradev-backend-upstream` (fork), built on `upstream/main` 427056c4, without `.scratch/`.
Not pushed upstream. Per `CONTRIBUTING.md` it must go through no-mistakes, which opens the PR itself:

```sh
no-mistakes init --fork-url git@github.com:robertrv/firstmate.git
git switch paradev-backend-upstream
git push no-mistakes
no-mistakes   # attach, watch findings, approve fixes; it opens the PR when green
```

## Title

Add experimental explicit-only Paradev runtime backend

## Body

Adds `paradev` as an experimental, explicit-only runtime backend. Paradev (formerly Superset) is a macOS desktop app where each task is a workspace with its own branch, worktree and terminals. With this backend each crewmate is a Paradev workspace, so the crew is visible in Paradev's sidebar, diff viewer and PR pane.

Design:

- Follows the Orca pattern: Paradev owns the worktree and Treehouse is not used. A small shared seam (`fm_backend_owns_worktree`, `fm_backend_worktree_id_key`, `fm_backend_worktree_owner_label`) replaces the `= orca` checks in spawn, teardown, watch and bootstrap, so Orca and Paradev share the abort cleanup, recovery record, worktree path-match gate and teardown order. Orca's behaviour and messages are unchanged.
- Never auto-detected: Paradev exports `SUPERSET_WORKSPACE_ID` into every terminal it opens, so that variable cannot mean the user chose Paradev.
- Every created workspace gets a display-only `firstmate` tag, which groups the crew into a sidebar folder. Ownership and recovery come only from ids recorded in task metadata. Nothing lists or acts on workspaces by tag.
- Teardown runs the normal landing checks. It then verifies the recorded workspace id still resolves to the recorded worktree path, closes the terminal and proves it gone, and only then archives. Paradev's `workspaces archive` deletes the worktree folder, uncommitted changes included.
- Readiness is a real `superset projects list --local --json` call, because `superset status` can report a stale host while every other call works.

Also included, as separate commits, two pre-existing macOS test failures hit while verifying:

- `bin/fm-hermes-lib.sh`: the bash 3.2 empty-array `set -u` bug, at a call site that #8399 did not cover (`tests/fm-control.test.sh`).
- `tests/fm-bootstrap.test.sh`: three cases called bare GNU `timeout`, which macOS does not ship; they now use `fm_run_timed`.

Verification (macOS, superset 2.0.8):

- `bin/fm-lint.sh` clean; `bin/fm-doc-audience-check.sh` ok.
- `tests/fm-backend-paradev.test.sh` (fake `superset`), plus Paradev cases in `fm-backend`, `fm-teardown-endpoint-safety`, `fm-bootstrap`, `fm-busy-state` and `fm-test-run` tests; Orca suites unchanged and green.
- Opt-in live smoke `tests/fm-backend-paradev-smoke.test.sh` (`FM_PARADEV_SMOKE_PROJECT=<project id>`): two crewmates, peek, send, keys, teardown, with cleanup even on failure. It records a gate skip on CI.
- Dogfood: a Claude first mate in a Paradev workspace ran two parallel ship tasks on a local-only project. Both crewmates appeared as tagged workspaces, landed, and were torn down with no lost work.

Known limits (documented in `docs/paradev-backend.md`): macOS only; no secondmates; no `fm-control exit/relaunch`; native busy state needs Paradev's Claude hooks in the active `CLAUDE_CONFIG_DIR` (upstream Paradev issue ParadevAI/Paradev#9724).
