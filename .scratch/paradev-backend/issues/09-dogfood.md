# 09 Dogfood on a small repo with two parallel tasks

Status: resolved
Type: task
Blocked by: 06, 07, 08

Run the first mate in a Paradev workspace named Captain, give it two parallel tasks on a small repo (for example `xeger`). Check: crewmates appear in one sidebar folder, the watcher wakes correctly, teardown loses no work, and the user never leaves the app.

## Comments

### 2026-11-14: dogfood run

Setup: this fork's checkout was the Firstmate home with `config/backend` = `paradev`. The first mate (Claude) ran in a Paradev workspace named **Captain** (project `firstmate`, `--checkout local`, tag `firstmate`). The first mate created the project `fm-dogfood` itself as `local-only` (no remote, no GitHub calls). The captain then seeded it with a small Python package and tests, and registered it with `superset projects create --import`.

Two parallel ship tasks were given in one message: `divide()` in `dogfood/calc.py`, and a new `dogfood/text.py` with `slugify()`.

Results:
- Both crewmates spawned as Paradev workspaces (`fm-dogfood-divide`, `fm-dogfood-slugify`), tagged `firstmate`, on Paradev-owned worktrees under `~/.paradev/worktrees/fm-dogfood/`. Task metadata recorded `backend=paradev`, `paradev_workspace_id`, `terminal=<ws>:<term>` and `paradev_project_id` as designed.
- Both delivered and reported ready in about 2 minutes. The watcher woke the first mate for each, and it verified tests itself and asked for merge approval.
- Landing: divide landed by local fast-forward. Slugify needed a rebase onto the new main (see gap 1), then landed through a guarded follow-up task. Final local main: seed, divide, slugify. All 10 tests pass, README unchanged.
- Teardown: all three crew workspaces were archived, worktrees and branches removed, and task records closed. No work was lost.

Findings:
1. **Firstmate core gap, not Paradev-specific: a rebased local-only ship cannot land under its own task.** The merge guard only accepts the head from the worker's first ready report, so the second of two parallel local-only ships, once rebased onto the new main, is refused. The workaround was a follow-up task pointing at the rebased commit. Worth an upstream issue with a backend-independent repro (tmux).
2. **Each teardown force-stopped one of three processes found in the worktree.** `bin/fm-teardown.sh` reaps worktree processes (TERM, then KILL after about 1 second) before it closes the endpoint, for every backend. So it signals the crewmate's own live shell and harness, and an interactive zsh ignores SIGTERM. Only pids are logged, so this is inferred from code order and shell behaviour, not proved. A separate repro showed `terminals close` alone leaves no process in the worktree. Not a Paradev adapter defect.
3. **Sidebar folders are per project.** The Captain (project `firstmate`) and the crew (project `fm-dogfood`) appear in two different `firstmate` folders. The "one folder" UX target holds per project only. Paradev has no cross-project folder.
4. Two pauses of about 15 minutes were macOS maintenance sleep (`pmset -g log`), not the product.
5. Do not archive the Captain workspace: it is `--checkout local` on this home's checkout, and archive behaviour for a local workspace was not verified.
