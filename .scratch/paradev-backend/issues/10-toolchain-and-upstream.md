# 10 Toolchain and upstream path

Status: resolved
Type: task

- Install Firstmate's toolchain per `docs/configuration.md#toolchain` (Treehouse is missing on this machine).
- Install `no-mistakes` v1.46.0+ and run `no-mistakes init --fork-url git@github.com:robertrv/firstmate.git` before the first upstream PR (`CONTRIBUTING.md`).
- The upstream PR branch must be cut from `upstream/main` and must not carry `.scratch/`: this planning folder lives only on the fork's `paradev-backend` branch.

## Comments

2026-11-14: no-mistakes v1.106.1 installed (binary `~/.no-mistakes/bin`, linked from `/opt/homebrew/bin` because `/usr/local/bin` needs sudo; the installer also started its daemon). Not yet initialised for this repo: run `no-mistakes init --fork-url git@github.com:robertrv/firstmate.git` when the upstream PR branch is cut.
Treehouse is not needed: `backend=paradev` follows the Orca pattern, and `docs/configuration.md` "Toolchain" lists Treehouse only for session-provider backends. Still missing from the universal toolchain, needed before dogfooding (09): `gh-axi`, `chrome-devtools-axi`, `tasks-axi`, `quota-axi` (`npm install -g <tool>`; the first two also run `<tool> setup hooks`).

2026-11-14: Upstream branch prepared, not published.
- Local branch `paradev-backend-upstream` in `~/.paradev/projects/firstmate`, on `upstream/main` 427056c4 (upstream had not moved since the fork), with no `.scratch/`. It holds three commits: the Hermes bash 3.2 fix (a call site #8399 did not cover), the portable-timeout fix in `tests/fm-bootstrap.test.sh`, and the backend itself (31 files, +2583/-202 in total).
- Verified on that branch: `bin/fm-lint.sh` clean, `bin/fm-doc-audience-check.sh` ok, `bin/fm-test-run.sh --check-coverage` ok (342 scripts), and 12 affected suites green (395 ok, 0 not ok, no gate skips, live smoke included).
- Toolchain now installed: no-mistakes 1.106.1, ShellCheck 0.11.0, actionlint 1.7.12, gh-axi, chrome-devtools-axi, tasks-axi, quota-axi. The axi `setup hooks` step was skipped on purpose (it writes only to `~/.claude`).
- Remaining human step: publish through no-mistakes, which opens the PR against kunchenguid/firstmate. Commands and PR text: `../upstream-pr-draft.md`. Separate upstream issue for the dogfood rebase gap: `../upstream-issue-draft-rebase-gap.md` (needs a tmux repro first).

2026-11-14: Published through no-mistakes (run 01M7CP5KN0BQNNAJ0BFHC54VPG): https://github.com/kunchenguid/firstmate/pull/8471, from the fork branch `paradev-backend-upstream`.
- The pipeline added four fix commits on top of the three originals: teardown of terminal-less spawn-recovery records (review), a strict id/path check before any app-owned archive or release plus per-call timeouts in the smoke test (Greptile), a retry marker after a successful archive (Greptile), and moving that marker out of the task record into the sidecar `state/<id>.app-worktree-released` so PR-poll receipt validation still parses the record (Greptile).
- Head 8758224 verified locally: lint clean; paradev 50, orca 55, teardown-endpoint-safety 30 and teardown 166 ok with 0 not ok; live Paradev smoke all pass. Evidence posted as a PR comment.
- Greptile passes. The GitHub workflows are held for first-time-contributor approval, and "Full suite" needs the `full-ci` label: both are maintainer actions. The no-mistakes CI step keeps watching and will handle any findings once the workflows run (`no-mistakes axi status` from `~/dev/firstmate-upstream`).
- Still open, separately: the rebase-gap upstream issue (`../upstream-issue-draft-rebase-gap.md`), which needs a tmux repro before filing.
