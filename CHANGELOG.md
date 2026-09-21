# Changelog

What changed in each skram release, for someone running the binary. Versions
follow semver.

## [0.13.0] — 2026-09-21

First public release.

- `skram`, a single binary for macOS and Linux (amd64 and arm64): one job
  queue that you and your coding agents share, one job at a time per shared
  resource, with every run logged.
- `skram init --detect` registers a repo's ops script, Makefile, Taskfile, or
  justfile as a project; `run`, `wait`, `status`, `logs`, `explain`, and
  `discover` all take `--json`.
- An MCP server over the same queue, and `skram agent install-rules` to tell
  the agents in a checkout how to use it.
- A TUI, a dashboard with live logs, job-end notifications, a Prometheus
  exporter, and a Docker disk reaper.
- `install.sh` downloads the latest release for your machine, verifies it
  against `checksums.txt`, and installs to `~/.local/bin` (`VERSION`,
  `BIN_DIR`). No GitHub login is needed. Shell completions and the herdr
  plugin manifest are in each archive.
- `THIRD_PARTY_NOTICES.md`, in the repository and in every release archive.
