# 10 Toolchain and upstream path

Status: ready-for-human
Type: task

- Install Firstmate's toolchain per `docs/configuration.md#toolchain` (Treehouse is missing on this machine).
- Install `no-mistakes` v1.46.0+ and run `no-mistakes init --fork-url git@github.com:robertrv/firstmate.git` before the first upstream PR (`CONTRIBUTING.md`).
- The upstream PR branch must be cut from `upstream/main` and must not carry `.scratch/`: this planning folder lives only on the fork's `paradev-backend` branch.

## Comments

2026-11-14: no-mistakes v1.106.1 installed (binary `~/.no-mistakes/bin`, linked from `/opt/homebrew/bin` because `/usr/local/bin` needs sudo; the installer also started its daemon). Not yet initialised for this repo: run `no-mistakes init --fork-url git@github.com:robertrv/firstmate.git` when the upstream PR branch is cut.
Treehouse is not needed: `backend=paradev` follows the Orca pattern, and `docs/configuration.md` "Toolchain" lists Treehouse only for session-provider backends. Still missing from the universal toolchain, needed before dogfooding (09): `gh-axi`, `chrome-devtools-axi`, `tasks-axi`, `quota-axi` (`npm install -g <tool>`; the first two also run `<tool> setup hooks`).
