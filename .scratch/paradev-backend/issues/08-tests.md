# 08 Tests: unit and smoke

Status: needs-triage
Type: task
Blocked by: 06

Copy the pattern of `tests/fm-backend-orca.test.sh` (stubbed `superset` on PATH) and add a live smoke test: spawn two crewmates, peek, send, teardown. Wire them into the CI lanes the way other backends are wired.
