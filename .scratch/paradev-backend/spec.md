# Firstmate on Paradev: backend adapter spec

> Status (2026-11-13): spike done. The UNVERIFIED list below is answered in
> [`spike-findings.md`](spike-findings.md). Two results change this spec:
> the CLI has native agent state, and `workspaces archive` deletes a worktree
> folder even when Paradev did not create it. The second one reopens the
> "who owns the worktree" decision (see [`issues/04`](issues/04-decide-worktree-ownership.md)).
> Work is tracked one ticket per file in [`issues/`](issues/).

## Goal

Talk to ONE agent (the Firstmate first mate), while each crewmate runs as a visible workspace inside Paradev (formerly Superset) instead of a tmux window. No second UI: Paradev's sidebar, diff viewer and PR pane are the crew view.

## Terminology

- **Crewmate** (Firstmate): one worker agent in its own git worktree, doing one task.
- **Workspace** (Paradev): one task item in the sidebar. It has its own branch and worktree, and hosts terminals and agent sessions.
- **Project** (Paradev): a repository checked out on a host. Workspaces branch off it.
- **Backend** (Firstmate): the session provider. Today tmux (default), herdr, zellij, cmux, Orca and others. Goal: add `paradev`.
- **Mapping**: one crewmate = one Paradev workspace, tagged so all crewmates of a first mate group into one sidebar folder.

## Verified facts (from docs read)

- Firstmate is a repo of instructions plus bash scripts. Backend adapters live in `bin/backends/` (for example `herdr.sh`), behind the dispatcher `bin/fm-backend.sh`. Backend is selected with `config/backend` or `FM_BACKEND`.
- The zellij backend was added as a session-provider only. Treehouse stays the worktree provider. Task metadata records backend plus endpoint ids.
- Firstmate's watcher needs per-task states: busy, idle, dead, missing, plus screen capture and text injection. Herdr provides these natively. Tmux and zellij fall back to polling and screen hashing.
- Paradev is MIT-licensed and public on GitHub, so it can be forked.
- Paradev CLI is `superset` (the Paradev rename keeps this command). It has `--json` and `--quiet` output, a `schema` command that prints every command as JSON, and `$SUPERSET_WORKSPACE_ID` is set inside workspaces.
- `superset workspaces create --project <id> --name <n> --branch <b> --local --agent claude --prompt "..." --tag <t>`: creates a workspace with a worktree and spawns an agent. `--agent` requires `--prompt`. `--tag` is repeatable and groups workspaces into a sidebar folder.
- `superset agents create --workspace <id> --agent <preset> --prompt ...`: starts an agent in a fresh terminal session. Supports `--resume-session`, `--fork-session`, `--from-terminal`. With `--json` it returns `kind` and `sessionId`.
- `superset workspaces get <id> --field worktreePath`, `workspaces list --tag`, `workspaces archive <ids>`, `workspaces update --tag/--name`.
- `superset worktrees open --project <id> --path <dir>` or `--branch <b>`: adopts an EXISTING worktree as a workspace. This is the key to reusing Firstmate's own worktrees.
- Deep links open a specific session: `superset://v2-workspace/<wsId>?terminalId=<id>`.
- MCP server and TypeScript SDK exist for the same operations.

## UNVERIFIED (must be checked first)

1. The `terminals` command group (read output, send input, kill). The docs page was fetched only up to the `schema` section. Run `superset terminals --help` and `superset schema | jq` to confirm.
2. Whether the CLI exposes per-agent status (busy, idle, blocked, dead). The app shows it in the sidebar. Check `superset agents`, `superset snapshot`, `superset panes`, and the MCP tool list.
3. Whether a workspace created without a prompt gives an idle terminal that Firstmate can type into later.
4. Whether tags work on workspaces created by hand (documented for automations and for `workspaces create --tag`).
5. Whether `--checkout worktree` honours a pre-made worktree, or whether `worktrees open` must be used.

## Architecture decision: who owns the worktree

Firstmate acquires worktrees through treehouse, and its cleanup and recovery logic assume that. Two options:

- **Option A (recommended for v1): Firstmate keeps treehouse.** The adapter then calls `superset worktrees open --project <id> --path <treehouse worktree>` to show it as a workspace. Minimal change to Firstmate core. Paradev is only the viewing and terminal layer.
- **Option B: Paradev owns worktrees.** The adapter calls `workspaces create --branch ...` and reads `worktreePath` back. Cleaner UX, but Firstmate's teardown and recovery logic needs rework.

Start with A. Move to B only if A's double bookkeeping hurts.

## Adapter contract to implement

Model it on `bin/backends/herdr.sh` and the zellij backend doc. Read both first, plus `docs/architecture.md` and `docs/configuration.md` (runtime backend section). Operations to map:

- **spawn**: create or adopt a workspace for the task, tag it (for example `firstmate`), start the crew agent with the task prompt, record endpoint ids (workspace id, session id, terminal id) in task metadata with `backend=paradev`.
- **send**: inject text or keys into the crewmate's terminal.
- **capture / peek**: read recent terminal output.
- **agent state**: return busy, idle, dead or missing. If the CLI cannot give native state, fall back to screen-hash polling as tmux does, and return `unknown` for busy-state like zellij does.
- **list-live**: workspaces carrying the firstmate tag.
- **teardown**: archive the workspace. Careful: archiving a worktree workspace removes its folder with uncommitted changes. Only archive after Firstmate's own landing checks pass, and under Option A use the equivalent of untracking instead if available.
- **preflight**: fail before creating anything if `superset` is missing, the host server is not running (`superset start --daemon`), or `jq` is missing.

## Setup requirements

- Paradev desktop app installed (macOS; Linux AppImage is experimental; no Windows).
- `superset auth login`, host server running, the target repo registered as a project (`superset projects create --import <path> --local` or `projects setup`), and the project id known.
- Firstmate cloned, plus its toolchain: bash, jq, git, `gh`, treehouse, and a primary harness such as Claude Code.
- Fork Firstmate (MIT). Paradev itself needs no changes for v1.

## Implementation plan

1. **Spike (read-only)**: run `superset schema`, `superset terminals --help`, `superset agents --help`. Write down the exact commands for read output, send input, and agent status. Resolve every UNVERIFIED item above.
2. **Manual proof**: in a shell, script by hand: create a workspace with an agent and a prompt, read its terminal, send it a follow-up, archive it. Confirm the workspace appears in the sidebar under the tag.
3. **Read Firstmate's backend code**: `bin/fm-backend.sh` dispatcher, `bin/backends/herdr.sh`, and the zellij PR (kunchenguid/firstmate pull 217) as the minimal template.
4. **Write `bin/backends/paradev.sh`** implementing the contract above, selected by `FM_BACKEND=paradev`. Add `docs/paradev-backend.md` mirroring the zellij doc.
5. **Mark it experimental and never auto-detected** (as zellij is), so it cannot hijack ambient sessions.
6. **Tests**: copy the pattern of the existing backend tests in the repo. Add a smoke test: spawn two crewmates, peek, send, teardown.
7. **Dogfood** on a small repo with two parallel tasks. Check: crewmates appear in one sidebar folder, the first mate's watcher wakes correctly, and teardown loses no work.

## UX target

You open Paradev and run Firstmate's first mate in one workspace (name it Captain). You say what you want. Crewmate workspaces appear in a tagged sidebar folder, each with its agent working. You review diffs and PRs in Paradev. You never leave the app.

## Risks

- Without native agent status, Firstmate's stuck-agent detection degrades to hash polling.
- Firstmate docs are large and evolving fast; the adapter may lag behind core changes.
- Archiving a workspace deletes its worktree folder. Guard against data loss.
- A hierarchy view (captain above crewmates) does not exist in Paradev. Tags and folders are the closest substitute.

## Fallback if the adapter proves too heavy

Skip Firstmate. Run a single captain agent in a Paradev workspace with the Paradev MCP server attached, and prompt it to create workspaces, launch agents, read terminals and collect results. You lose Firstmate's supervision logic but gain a working system in an afternoon.

## Sources

- Firstmate repo and docs: github.com/kunchenguid/firstmate (architecture.md, herdr-backend.md, zellij-backend.md, configuration.md)
- Paradev CLI reference: docs.paradev.com/cli/cli-reference
- Paradev orchestration and MCP: paradev.com/agent-orchestration and docs.paradev.com/mcp-server
