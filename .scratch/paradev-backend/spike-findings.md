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
   **Resolved (issue 03):** Paradev's Claude hooks live only in `~/.claude/settings.json`, but this
   machine exports `CLAUDE_CONFIG_DIR=~/.claude-anthropic`, so the hooks never load. Launching Claude
   with `--settings <hooks-only file>` gives all five states end to end, and sends submit. Filed
   upstream as ParadevAI/Paradev#9724. A send right after `agents create` can still race the launch
   command into the shell: wait for a state change before the first send.
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
| agent state | `agents explain ... --json` -> `.state` (map: working->busy, idle/blocked->idle, exited->dead, missing workspace/terminal->missing, starting->unknown); reliable once hooks load (issue 03); `starting` that lasts past launch means the hooks are missing, so report a setup error |
| list-live | `workspaces list --local --tag <tag> --json` (+ `snapshot --json` for terminal liveness) |
| teardown | after Firstmate landing checks: `terminals close`, then `workspaces archive <ws>` (B). Under A see issue 04 |
| focus/open | `workspaces open <ws>` or deep link `superset://v2-workspace/<ws>?terminalId=<t>` |

## Manual crewmate lifecycle (issue 02)

Date: 2026-11-13, after the issue 03 local hooks fix. Plain `workspaces create --agent claude`, no
`--settings` workaround. All calls take `--local --json`; ids below are shortened.

### Commands and JSON shapes

| Step | Command | Result shape |
|------|---------|--------------|
| spawn | `workspaces create --local --project <p> --name fm-spike-02a --branch spike/02a --skip-branch-prefix --tag fm-spike --agent claude --prompt "..." --json` | `{workspace: {id, projectId, hostId, name, branch, type: "worktree", ...}, terminals: [], agents: [{ok, kind: "terminal", sessionId, label: "Claude"}], alreadyExists, url, webUrl}`. Record `workspace.id` and `agents[0].sessionId` (the terminal id). `worktreePath` is not in this reply; `workspaces list/get` has it. Without `--skip-branch-prefix` the project branch prefix is prepended. |
| state | `agents explain --workspace <w> --terminal <t>` | `{terminalId, state, agentId, headline, because, details[], lastEventType, lastEventAt}` |
| wait | `agents wait --workspace <w> --terminal <t> --until idle --timeout N` | `{terminalId, state, lastEventType}` |
| transcript | `agents read --workspace <w> --terminal <t>` | `{harness: "claude", text: "User: ...\n\nAssistant: ..."}` |
| screen | `terminals read ... --max-lines N` | `{terminalId, cols, rows, text}`. `text` is the visible screen, bottom-padded with blank lines, so a small `--max-lines` can return only the empty input box. Use `agents read` for answers and `terminals read` for dialogs. |
| steer | `terminals send ... --text "..." --wait --until settled\|working --timeout N` | `{terminalId, submitted: true, state}` |
| keys | `terminals send ... --keys esc` / `ctrl+c` / `ctrl+c,ctrl+c` / `enter` | `{terminalId, keys: <count>}` (no state) |
| terminals | `terminals list --workspace <w>` | `{sessions: [{terminalId, workspaceId, createdAt, exited, exitCode, attached, title, customTitle}]}` |
| close | `terminals close --workspace <w> --terminal <t>` | `{terminalId, status: "disposed"}` |
| archive | `workspaces archive <w...>` | `{deleted: [ids], notFound: [], failed: [], warnings: []}` |
| list-live | `workspaces list --tag fm-spike` | array of workspace rows; `tags` is a string (`"fm-spike"`), plus `worktreePath`, `worktreeExists`, `archivedAt` |
| snapshot | `snapshot --workspace <w>` | `{workspaces: [{id, name, project, branch, terminals: [{id, title, agent: {id, state} \| null}], layout, layoutError}]}` |

### Observed timeline (two crewmates at once, same tag)

Both created in parallel (backgrounded `workspaces create`, 13 ms apart): no conflict, both listed
under `--tag fm-spike` at once, both reached `working` (`Start`) at the first poll and `idle`
(`Stop`) about 10 to 15 s later. Answers were correct (`## What it is`, the first Markdown heading of
README.md).

| Step | `agents explain` | `snapshot` | `workspaces list --tag` |
|------|------------------|------------|-------------------------|
| right after create | `working` / `Start` | terminal with `agent: {id: claude, state: working}`, title = prompt summary | both rows, `worktreeExists: true` |
| turn done | `idle` / `Stop` | `agent.state: idle` (title spinner glyph stays) | unchanged |
| follow-up `send --wait --until settled` | returned `{submitted: true, state: idle}` after 6 s; answer `435` matches `wc -l AGENTS.md` | | |
| `--keys esc` while working | `idle` / `Stop` within 2 s; screen `Interrupted · What should Claude do instead?` | `idle` | unchanged |
| `--keys ctrl+c` while working | same as Esc: `idle` / `Stop` within 2 s, `Interrupted` | `idle` | unchanged |
| `--keys ctrl+c,ctrl+c` at idle | still `idle`: Claude showed an exit dialog (background shell still running) | `idle` | unchanged |
| `--keys enter` on that dialog | `exited` ("the agent ended its own session") | terminal kept, `agent: null`, title = shell prompt; `terminals list` says `exited: false` (the shell lives on) | unchanged |
| `terminals close` (agent idle inside) | `exited` ("the terminal was closed on purpose") | `terminals: []` | unchanged, workspace and worktree stay |
| `terminals read` after close | error `Terminal session ... has ended`, exit 1 | | |
| `workspaces archive` (live shell inside) | error `Workspace not found on host`, exit 1 | error `... is not on this host` | row gone; worktree folder deleted |

### Findings for the adapter

1. **Native state works with plain `workspaces create --agent claude`** once the hooks are in the
   active config dir. `working` and `idle` arrived at the right moments for both crewmates.
2. **Esc and a single Ctrl-C both interrupt a turn** and read as `idle` / `Stop`, same as a finished
   turn. `because` does not tell them apart; an interrupted turn and a finished one look alike.
3. **Esc on a running Bash tool call backgrounded the command** (`1 shell still running`). The
   later double Ctrl-C then opened a "Exit and stop tasks / Move to background / Stay" dialog that
   native state reported as `idle`, not `blocked`. So `idle` does not guarantee a free input box: a
   non-permission dialog is invisible to native state. The watcher should still capture the screen
   when a crewmate sits `idle` without reporting. Interrupt with one key, not `ctrl+c,ctrl+c`.
4. **Agent exit leaves the terminal alive.** `agents create` runs Claude inside a shell, so after
   exit `terminals list` says `exited: false`, `snapshot` shows `agent: null`, and only
   `agents explain` says `exited`. Use `agents explain` (or `snapshot` `agent == null`) for "dead",
   not `terminals list.exited`. Claude prints its `--resume <session>` id on exit.
5. **Missing vs dead:** a closed terminal gives `explain: exited` (exit 0), a read error (exit 1).
   An unknown terminal id in a live workspace gives `state: "none"` with exit 0, not an error. An
   archived workspace gives exit 1 `Workspace not found on host` from every per-workspace call.
   Map: `none` or workspace-not-found -> missing; `exited` -> dead.
6. **Close vs archive:** `terminals close` kills the agent and the terminal only; the workspace,
   tag and worktree stay and `list --tag` still shows it. `workspaces archive` removes everything,
   including the worktree folder, with no warning even while a terminal is live, and drops the row
   from `list --tag` and `snapshot`. The branch survives archive and must be deleted separately.
7. **`snapshot` layout is empty** (`tabs: []`) for CLI-created workspaces nobody has opened in the
   app. Liveness comes from `terminals`, not `layout`.
