# 07 Write `docs/paradev-backend.md` and skill

Status: resolved
Type: task
Blocked by: 06

Mirror `docs/orca-backend.md`: setup, task shape and metadata, lifecycle and safety, active limits, regression entry points. Add a `firstmate-paradev` skill if the Orca pattern requires one, and the `docs/verification/runtime-backends.md` entry.

## Comments

### 2026-11-14 - docs and skill written (agent)

**What landed** (commit `8262fe5a` on `paradev/07-docs`):

- `docs/paradev-backend.md`, mirroring `docs/orca-backend.md`: Setup (desktop app and host service, `superset` + `jq`, no Treehouse, the clone must be a registered Paradev project at that exact path, explicit-only and why `SUPERSET_WORKSPACE_ID` cannot select it, readiness by a real `projects list` call not `status`, the display-only `firstmate` tag and the captain workspace, supported keys, launch-env allowlist caveat), Task shape and metadata (the keys ticket 06 actually records), Current lifecycle and safety (spawn, peek, send, a Native agent state subsection with the hooks / `CLAUDE_CONFIG_DIR` gap and upstream #9724, and a Cleanup subsection including the archive-deletes-the-worktree hazard), Active limits, Regression entry points.
- `docs/verification/runtime-backends.md`: a `## Paradev` section (superset 2.0.8, macOS 26.7.3; readiness, every accepted response shape, lifecycle facts table, regression run), a `paradev` row in "Endpoint close" plus why its close refuses under `--force`, Paradev in the force-exempt sentence, and `tests/fm-backend-paradev.test.sh` plus the Paradev fixture case in "Cleanup endpoint identity". No ids, paths, branches or chronology, per the file's preamble.
- `docs/configuration.md`: backend table row, Treehouse sentence, never-auto-detected and marker-leak sentences, accepted-backends and secondmate-refusal lists, a dependency-check bullet, a Task metadata bullet, the toolchain table row and `jq` / Treehouse sentences, the `FM_BACKEND` env comment.
- `docs/architecture.md`: adapter list, Paradev's native busy participation, secondmate sentence, a Paradev paragraph that also documents the shared worktree-owner seam (`fm_backend_owns_worktree`) ticket 06 introduced, the worktrees sentence.
- `README.md`: features (visible crew, disposable worktrees), "More backends" list, docs list entry.
- `docs/documentation-audiences.json`: the doc as `operator-current` and a README setup target, its owner pointer to the verification record, the skill as `agent-runtime`.

**Skill decision: added `.agents/skills/firstmate-paradev`, registered in `agent-skill-trigger-index`.** Orca's precedent alone would not justify it; three Paradev-specific facts do, and they are situational agent knowledge (coding-guidelines tier 2), not operator docs:
1. Paradev hands the operating agent a full scripting CLI (and MCP) for the very workspaces it supervises, and the most natural "cleanup" call, `workspaces archive`, deletes the worktree with uncommitted work and no warning. The skill's trigger includes "before running any raw `superset` command that could close, archive, or retag a workspace", which no doc can enforce.
2. The tag folder mixes the captain's own workspace with the crew, and the first mate usually runs inside a Paradev workspace, so `SUPERSET_WORKSPACE_ID` in its own env names the captain's workspace. An agent reasoning from the sidebar or its env would pick the wrong target.
3. A Claude stuck at native `starting` is a captain setup gap (hooks not in the active config dir) to report, not something to fix by editing local config.
Everything else in the skill is the Orca checklist adapted, and it points at `docs/paradev-backend.md` for the contract rather than restating it. Cost: one line in the trigger index; the body loads only when Paradev is in play.

Checked live (read-only) for the docs: this workspace's worktree is a linked git worktree sharing the project's common git dir (the Claude trust check's requirement); `workspaces get` has no `archivedAt` key; `superset status` still reports `running=false stale=true` while calls work.

**Quality bar.**
- `bin/fm-lint.sh`: clean (ShellCheck 0.11.0, actionlint 1.7.12).
- `bin/fm-test-run.sh --family paradev`: `total=1 failed=0`, 46 ok. `tests/fm-harness-adapter-references.test.sh`: passes.
- `bin/fm-doc-audience-check.sh` and `tests/fm-documentation-audiences.test.sh` **fail on this branch, pre-existing and not caused by this ticket**: every `.scratch/paradev-backend/*.md` is unclassified. Evidence: in throwaway worktrees with `.scratch/` removed from the index, both the base commit `16a2d7fd` (`ok surfaces=142 local_links=1248`) and this commit (`ok surfaces=144 local_links=1261`) pass, and both tests in the suite pass. `.scratch/` is deliberately not classified in the inventory because it must not reach upstream (ticket 10); the upstream PR branch drops it, which makes the check green.
- No `bin/` or `tests/` files changed.

**Left for other tickets.** Ticket 08: when the live smoke lands, add its command and dated result to the `## Paradev` verification section and to `docs/paradev-backend.md` "Regression entry points" (and Paradev to `FM_BACKEND_VISIBLE_CAPTURE` docs if that changes). Ticket 09: the dogfood should confirm the setup steps in `docs/paradev-backend.md` are complete from a fresh home.
