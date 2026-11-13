# 02 Manual proof: spawn, peek, steer, archive by hand

Status: ready-for-agent
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
