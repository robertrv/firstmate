# Spike findings: Paradev CLI surface for a Firstmate backend

Date: 2026-11-13. CLI: `superset v2.0.8` on macOS. Paradev project `firstmate`
(id `b9858b1a-100c-47b1-8304-a268c314eca6`) = this fork at `~/.paradev/projects/firstmate`.

Method: `superset schema` (full dump, read-only), `--help` per group, then a hand-run
proof on throwaway branches `spike/noprompt` and `spike/adopt` (both cleaned up).

## Answers to the spec's UNVERIFIED list

| # | Question | Answer | Evidence |
|---|----------|--------|----------|
| 1 | `terminals` group: read, send, kill | **Yes.** `terminals read --max-lines N`, `terminals send --text` / `--keys esc,enter,ctrl+c,...` / `--no-submit`, `terminals close`, `terminals create --command`, `terminals list`, `terminals wait --match <regex>` | schema; shell round trip `echo hello-from-fm` sent and read back |
| 2 | Native per-agent status | **Yes, in the CLI.** States: `starting`, `working`, `blocked`, `idle`, `exited`. `agents explain` (one-shot, JSON with `state`, `because`, `lastEventType`), `agents wait --until settled\|idle\|blocked\|working`, `terminals send --wait --until ...`, and `snapshot` (every workspace, terminal, `agent.state`). `agents read` returns the agent's own transcript. | schema; `agents explain` output |
| 3 | Workspace without prompt gives an idle terminal | **No.** `workspaces create` without `--agent` returns `terminals: []`. Firstmate must call `terminals create` (shell) or `agents create` itself. | proof |
| 4 | Tags on workspaces not created with `--tag` | **Yes.** `workspaces update <id> --tag fm-spike` on an adopted workspace, then `workspaces list --tag fm-spike` lists it. Note `--tag` on update *replaces* the tag set. | proof |
| 5 | `--checkout worktree` honours a pre-made worktree | Moot. `worktrees open --project <id> --path <dir>` adopts an existing worktree and is the right call. | proof |

## Findings that change the plan

1. **`workspaces archive` deletes the worktree folder even for an adopted worktree.**
   A worktree created by plain `git worktree add` outside Paradev, opened with
   `worktrees open`, holding an uncommitted file: `archive` returned `deleted` with
   no warning and the folder was gone. The branch survived. There is no "untrack"
   or "forget" command in the schema. Under Option A this means Paradev and
   Treehouse would both delete the same folder, and any archive is a data-loss
   hazard. See issue 04.
2. **Native state did not move past `starting` for a Claude agent in this run.**
   `agents create --agent claude --prompt ...` launched Claude (its terminal title
   became the prompt summary), but `agents explain` kept reporting
   `starting` / `lastEventType: Attached`, `agents wait --until settled` timed out
   after 120s, `agents read` said "No transcript", and a follow-up
   `terminals send --text` stayed unsubmitted in Claude's input box (an extra
   `--keys enter` did not submit it either). Cause unknown: candidates are the
   host service state (below), the Claude hook wiring Paradev relies on, or
   running nested inside another Claude session. See issue 03. Until this is
   understood, native state cannot be trusted as the watcher's only signal.
3. **`superset status` reported `running: false, stale: true`** while every other
   command (snapshot, create, send, archive) worked. Preflight must probe with a
   real cheap call (for example `superset projects list --local --json`) rather
   than trust `status.running`.
4. **Inside an agent environment `--json` is auto-on**, and `workspaces get --field worktreePath`
   then printed the whole JSON object rather than the raw field. The adapter
   should always pass `--json` and extract with `jq`, never rely on `--field` or
   human output.
5. **`agents create` returns `{kind: "terminal", sessionId}` and that `sessionId` is the
   terminal id** used by `terminals read/send/close` and `agents explain/wait`.
   One id to record per crewmate, plus the workspace id.
6. **Firstmate already has an app-owns-the-worktree backend: Orca** (`bin/backends/orca.sh`,
   `docs/orca-backend.md`). Orca owns the worktree and terminal, `fm-spawn.sh`
   skips Treehouse, and cleanup validates a recorded worktree identity before
   asking the app to release it. That is exactly the shape of Option B, so
   Option B is not the "rework Firstmate core" path the spec assumed; it is a
   copy of an existing pattern.
7. **Toolchain gap on this machine:** `treehouse` is not installed (needed for Option A
   and for upstream Firstmate in general), and `no-mistakes` is required to
   raise the upstream PR (`CONTRIBUTING.md`).

## Operation map (draft, for issue 06)

| Adapter op | Paradev call |
|------------|--------------|
| preflight | `command -v superset jq`; `superset projects list --local --json` succeeds; project id resolves |
| spawn (B) | `workspaces create --local --project <id> --name fm-<task> --branch <b> --tag firstmate[-<home>] --json` then `agents create --local --workspace <ws> --agent <preset> --prompt <brief> --json` -> record `workspace`, `terminal`=sessionId, `worktree` |
| spawn (A) | `worktrees open --local --project <id> --path <treehouse wt> --json`, `workspaces update <ws> --tag ...`, then `agents create` |
| send | `terminals send --local --workspace <ws> --terminal <t> --text ...` / `--keys ...` |
| capture/peek | `terminals read --local --workspace <ws> --terminal <t> --max-lines N --json` -> `.text` |
| agent state | `agents explain ... --json` -> `.state` (map: working->busy, idle/blocked->idle, exited->dead, missing workspace/terminal->missing, starting->unknown); hash-poll fallback until issue 03 is resolved |
| list-live | `workspaces list --local --tag <tag> --json` (+ `snapshot --json` for terminal liveness) |
| teardown | after Firstmate landing checks: `terminals close`, then `workspaces archive <ws>` (B). Under A see issue 04 |
| focus/open | `workspaces open <ws>` or deep link `superset://v2-workspace/<ws>?terminalId=<t>` |
