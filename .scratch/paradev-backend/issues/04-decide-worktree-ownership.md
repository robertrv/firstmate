# 04 Decide: who owns the crewmate worktree (Option A vs B)

Status: needs-triage
Type: grilling

The spec picked Option A (Treehouse owns, Paradev adopts via `worktrees open`). The spike found:

- `workspaces archive` deletes an adopted worktree folder, uncommitted changes included, with no warning; there is no untrack command. Under A every teardown path must never call archive, or must call it only after Treehouse has already released the worktree (behaviour then untested), and stale workspaces otherwise pile up in the sidebar.
- Firstmate already ships an app-owns-the-worktree backend (Orca): `fm-spawn.sh` skips Treehouse, cleanup verifies a recorded worktree identity and asks the app to release it. Option B is a copy of that pattern, not a core rework.
- Treehouse is not installed on this machine.

Recommendation: Option B, modelled on `bin/backends/orca.sh`, with Firstmate's landing checks run before `archive`. Record the decision as an ADR once agreed.

## Open questions for the captain

1. Accept B for v1?
2. Tag scheme: one `firstmate` tag, or `firstmate-<home>` per first mate home?
3. Should the Captain workspace itself be tagged and excluded from list-live?
