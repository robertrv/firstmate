# 03 Diagnose: Claude agent stuck in `starting`, sends not submitted

Status: resolved
Type: research

During the spike, `agents create --agent claude --prompt ...` launched Claude but `agents explain` never left `starting` (`lastEventType: Attached`), `agents wait` timed out, `agents read` had no transcript, and `terminals send --text` (with and without an extra `--keys enter`) left the text in Claude's input box.

Reproduce first, the way a user would: create the workspace and agent from the Paradev desktop app, then from the CLI from a plain terminal (not nested inside a Claude session). Then check:

- host service health (`superset status` said `running: false, stale: true` while commands worked; try `superset start --daemon`)
- Paradev's Claude hook wiring (`~/.paradev/agent-hooks.json`, `~/.paradev/hooks`) and whether the launched Claude loads it
- whether `--no-submit` + `--keys enter` or a delay before submit behaves differently

## Acceptance

Native state transitions observed end to end, or a written conclusion that it is unreliable and the adapter must fall back to screen-hash polling (zellij style). If it is a Paradev bug, file it upstream with a minimal repro.

## Comments

2026-11-13: Root cause found, native state verified end to end, upstream bug filed: https://github.com/ParadevAI/Paradev/issues/9724

**Cause: Paradev's Claude hooks are not in the config dir Claude reads.**
`~/.zshrc` exports `CLAUDE_CONFIG_DIR=$HOME/.claude-anthropic`. Paradev writes its managed hooks
(`notify.sh` on SessionStart, UserPromptSubmit, Stop, PermissionRequest, ...) only to
`~/.claude/settings.json`, plus `<dir>/settings.json` for accounts added in its Usage tab
(`ensureClaudeManagedHooksAt` in `packages/agent-setup`). A config dir exported by hand wins at
launch but never gets the hooks, so no lifecycle hook reaches the host and `agents explain` stays at
`starting` / `Attached` (the `Attached` event comes from Paradev's launch, not from Claude).
Even this ticket's own Claude session, started from the Paradev desktop app, reported `starting`
while it was running tools, so the app path and the CLI path behave the same.

Ruled out:
- Host service health: `status` still says `running: false, stale: true` (pid alive), and every
  command, hook delivery included, works. Unrelated to this; preflight should keep using a real call.
- Nesting inside another Claude session: the launched `claude` process has no `CLAUDECODE` /
  session variables (checked with `ps eww`). `agents create` is a host call, so the caller's env does
  not reach the agent.

**A/B (same workspace, same `/opt/homebrew/bin/claude`, same `CLAUDE_CONFIG_DIR`):**
- A, `agents create --agent claude --prompt "Reply with exactly the word PONG..."`: Claude printed
  PONG and went idle, `agents explain` stayed `starting` / `Attached` for 30s+.
- B, `terminals create --command "claude --settings <hooks-only.json> '...'"`, where
  `hooks-only.json` is `jq '{hooks}' ~/.claude/settings.json`: `working` (`Start`) -> `idle`
  (`Stop`) within 10s. Loading the whole `~/.claude/settings.json` instead also pulled in that
  profile's Bedrock model and env, and the turn failed (`Failed`), so only the `hooks` key must go in.
- With B: `terminals send --text ... --wait --until settled` returned `{submitted: true, state: idle}`
  after the turn; `agents wait --until idle` works; `agents read` returns `{harness, text}` with the
  full User/Assistant transcript; a `--permission-mode default` run showed `blocked`
  (`PermissionRequest`), Esc returned it to `idle`, `/exit` gave `exited` ("the agent ended its own
  session"). All five native states observed.

**Sends:** the spike's "text left unsubmitted in the input box" did not reproduce: a send after idle,
a send 1.5s after launch (during Claude's boot), and two sends during a working turn were all
submitted and queued. One real failure found instead: a send issued right after `agents create`
returns lands in zsh *before* the launch command line runs, corrupting it
(`zsh: command not found: Now`), and Claude never starts although `send` says `submitted: true`.
Most likely the spike hit a timing variant of this. `--no-submit` + `--keys enter` was not needed.

**Conclusion for the adapter (issue 06):** native state is reliable once the hooks load. Do not
fall back to zellij-style hash polling as the main signal.
- Launch Claude crewmates so the hooks always load, whatever `CLAUDE_CONFIG_DIR` is: either
  `terminals create --command "claude --settings <paradev-hooks.json> ..."` with a hooks-only file
  Firstmate builds from `~/.claude/settings.json`, or require the captain to add the exported config
  dir as a Paradev Usage-tab account (which installs the hooks there). Until upstream #9724 is fixed,
  the first is the one that needs no captain setup. It still needs a check in issue 06 that a
  `terminals create` agent shows in the sidebar like an `agents create` one.
- Before the first `send`, wait for `agents wait --until working|settled` (the initial prompt is
  passed on the command line, so the first hook arrives at once). Never send while the state is
  still `starting`.
- Health check after spawn: if the state is still `starting` N seconds after launch, the hooks did
  not load. Report it as a setup error rather than silently polling screen hashes.
- No local config was changed. Throwaway workspace `fm-spike-03a` (tag `fm-spike`) archived and its
  branch deleted.
