# 03 Diagnose: Claude agent stuck in `starting`, sends not submitted

Status: ready-for-agent
Type: research

During the spike, `agents create --agent claude --prompt ...` launched Claude but `agents explain` never left `starting` (`lastEventType: Attached`), `agents wait` timed out, `agents read` had no transcript, and `terminals send --text` (with and without an extra `--keys enter`) left the text in Claude's input box.

Reproduce first, the way a user would: create the workspace and agent from the Paradev desktop app, then from the CLI from a plain terminal (not nested inside a Claude session). Then check:

- host service health (`superset status` said `running: false, stale: true` while commands worked; try `superset start --daemon`)
- Paradev's Claude hook wiring (`~/.paradev/agent-hooks.json`, `~/.paradev/hooks`) and whether the launched Claude loads it
- whether `--no-submit` + `--keys enter` or a delay before submit behaves differently

## Acceptance

Native state transitions observed end to end, or a written conclusion that it is unreliable and the adapter must fall back to screen-hash polling (zellij style). If it is a Paradev bug, file it upstream with a minimal repro.
