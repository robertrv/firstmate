# 04 Decide: who owns the crewmate worktree (Option A vs B)

Status: resolved
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

## Decision (2026-11-13)

1. **Option B.** Paradev owns crewmate worktrees, modelled on the Orca backend. Treehouse is not used for `backend=paradev`. Teardown runs Firstmate's landing checks, verifies the recorded workspace id and worktree path, then closes the terminal and archives.
2. **One tag, `firstmate`, for every workspace Firstmate creates.** The tag is display only: it files workspaces into one sidebar folder. Ownership never comes from the tag. Every operation and recovery path uses the workspace and terminal ids recorded in task metadata, as Firstmate already requires ("never sweep a shared endpoint namespace"). Home separation, if secondmates are ever supported, comes from home-scoped workspace names via `bin/fm-backend-hometag-lib.sh` (as zellij does), not from extra tags.
3. **The Captain workspace may carry the `firstmate` tag too**, so the whole crew sits in one folder. This is safe because list-live and recovery are metadata-driven: an untracked tagged workspace (the Captain, or one the user tags by hand) is never treated as a crewmate, so it is never steered or archived. Untagged workspaces in the same project are likewise ignored.
4. Secondmate spawns are out of scope for v1 (Orca precedent).
