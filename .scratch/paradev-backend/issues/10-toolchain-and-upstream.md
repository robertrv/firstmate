# 10 Toolchain and upstream path

Status: ready-for-human
Type: task

- Install Firstmate's toolchain per `docs/configuration.md#toolchain` (Treehouse is missing on this machine).
- Install `no-mistakes` v1.46.0+ and run `no-mistakes init --fork-url git@github.com:robertrv/firstmate.git` before the first upstream PR (`CONTRIBUTING.md`).
- The upstream PR branch must be cut from `upstream/main` and must not carry `.scratch/`: this planning folder lives only on the fork's `paradev-backend` branch.
