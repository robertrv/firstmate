# 05 Read Firstmate's backend seams

Status: ready-for-agent
Type: research

Read `bin/fm-backend.sh` (dispatcher), `bin/backends/orca.sh` + `docs/orca-backend.md` (app-owned worktree, closest template for Option B), `bin/backends/zellij.sh` + `docs/zellij-backend.md` (explicit-only, hash-poll fallback), `docs/architecture.md`, `docs/configuration.md` "Runtime backend", and `.agents/skills/firstmate-orca`.

## Acceptance

A short note appended under `## Comments` listing: every function a backend must implement, how a backend is registered as explicit-only, where metadata keys are declared, which tests every backend must have (`tests/fm-backend-orca.test.sh`, `tests/fm-backend.test.sh`, `tests/fm-teardown-endpoint-safety.test.sh`, ...), and how `docs/verification/runtime-backends.md` is expected to be filled.
