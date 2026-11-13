# 06 Implement `bin/backends/paradev.sh`

Status: needs-triage
Type: task
Blocked by: 03, 04, 05

Implement the adapter contract from `../spec.md` using the operation map in `../spike-findings.md`. Selected by `FM_BACKEND=paradev` or `config/backend`, explicit-only, never auto-detected, macOS only. Always pass `--json --local` and parse with `jq`.

## Acceptance

- spawn / send / peek / state / list-live / teardown / preflight implemented.
- Teardown refuses unless Firstmate's landing checks pass, and verifies the recorded workspace id and worktree path before archive.
- Metadata: `backend=paradev`, `workspace=`, `terminal=`, `worktree=`, `paradev_project=`.
