# Changelog

What changed in each skram release, for someone running the binary. Versions
follow semver.

## [Unreleased]

## [0.14.0] — 2026-09-21

- Two `skram run` calls issued at the same instant on an idle queue
  could each start a processor and run two jobs on one resource at once. The
  processor now holds `processor.lock` for its life and a second one exits,
  and a lane stays taken while a job an earlier processor started is still
  running.
- Documentation, in `docs/`: a configuration reference for every key, how the
  queue works (jobs on disk, lanes, hold against drain against kill, estimates,
  exit codes), agent setup and the MCP server's tools, herdr, notifications,
  troubleshooting with every `SKRAM_*` variable, and an FAQ. Every command a
  user can see, every variable, and every config key is in them; a release
  cannot be tagged otherwise.
- The README opens with a recording of two callers sharing one queue.
- `skram --help` no longer lists the `lore`, `spec`, `ticket`, `vault`, and
  `tunnel` redirects; typing one still prints the replacement command.
- `skram init`'s starter file shows only keys the loader accepts, and the
  errors for removed keys point at `docs/agent-setup.md`.

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
