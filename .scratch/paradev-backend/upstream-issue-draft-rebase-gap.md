# Draft issue (not filed): a rebased local-only ship cannot land under its own task

Status: draft. Before filing, reproduce on the default `tmux` backend to show it is backend-independent, and cite the exact guard in `bin/` (found during the Paradev dogfood; see `issues/09-dogfood.md`).

## What happens

Two ship tasks run in parallel on a `local-only` project and both report ready. The first one lands by local fast-forward. The second must be rebased onto the new main to stay a fast-forward. After the worker rebases (new head commit), the guarded local merge refuses it, because the merge guard only accepts the head recorded in the worker's first ready report (the pre-rebase commit).

## Workaround used

The first mate opened a follow-up task whose branch points at the rebased commit and landed that through the normal guarded merge. The original task then counted as landed and both cleaned up normally. The alternative, a manual fast-forward, bypasses the guard and its records.

## Expected

A worker's re-reported ready head after a rebase (same task, same content plus the new base) can land under the original task through the guarded path, for example by accepting a fresh ready report that supersedes the first.
