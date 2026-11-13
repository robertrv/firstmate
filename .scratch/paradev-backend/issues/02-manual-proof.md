# 02 Manual proof: spawn, peek, steer, archive by hand

Status: claimed
Type: task
Blocked by: 03

Script by hand, in a shell, the full crewmate lifecycle against project `firstmate` (`b9858b1a-100c-47b1-8304-a268c314eca6`):
create a tagged workspace with a Claude agent and a real prompt, read its terminal, send a follow-up that gets submitted and answered, watch native state go `working -> idle`, close the terminal, archive.

## Acceptance

- Workspace shows in the sidebar under the tag folder.
- Follow-up `terminals send` is submitted and the agent answers.
- `agents explain` reports at least `working` and `idle` at the right moments.
- Exact commands and JSON shapes recorded in `../spike-findings.md`.

## Comments

2026-11-13: Shell-terminal half done (create, send, read, tag, archive). Agent half blocked by 03.

2026-11-13: 03 resolved. Unblocked: launch Claude with Paradev's hooks loaded (see 03's comments; `--settings <hooks-only file>`) and wait for a state change before the first send.

2026-11-13: Full agent lifecycle done by hand with plain `workspaces create --agent claude` (hooks now load). Commands, JSON shapes, a per-step table of `agents explain` / `snapshot` / `workspaces list --tag`, and adapter findings are in `../spike-findings.md` under "Manual crewmate lifecycle (issue 02)".

Acceptance:
- Follow-up `terminals send --wait --until settled` returned `submitted: true, state: idle` and the agent answered correctly (`435`, the line count of AGENTS.md). Met.
- `agents explain` reported `working` then `idle` at the right moments, for two crewmates created in parallel under the same tag. Met.
- Commands and JSON shapes recorded. Met.
- Sidebar folder: needs human eyes, not yet confirmed. Workspace **`fm-spike-sidebar-check`** (id `1a4e0322-c99c-4865-ba99-c1157ffb24d6`, branch `spike/sidebar-check`, tag `fm-spike`, Claude agent idle after replying READY) is left open for the captain to check that it shows in the Paradev sidebar under an `fm-spike` folder. It will be archived and its branch deleted only when the captain says so.

Also covered: Esc and a single Ctrl-C both interrupt a working turn and read as `idle` / `Stop`; a non-permission Claude dialog (exit confirmation) reads as `idle`, not `blocked`; agent exit leaves the shell terminal alive (`snapshot` `agent: null`, `explain` `exited`); `terminals close` keeps the workspace, `workspaces archive` deletes the worktree folder even with a live terminal and keeps the branch.

Cleanup: `fm-spike-02a` and `fm-spike-02b` archived, branches `spike/02a` and `spike/02b` deleted (never pushed). No local Claude or Paradev configuration changed. No send was issued before an agent left `starting`.

Status stays `claimed` until the sidebar check is confirmed.
