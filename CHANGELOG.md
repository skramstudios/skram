# Changelog

What changed in each skram release, for someone running the binary. Versions
follow semver.

## [Unreleased]

## [0.16.0] — 2026-09-22

- Run a project's targets in any checkout of its repo: a worktree, a
  worktree nested inside the checkout, or a second clone. From inside one,
  `skram run <target>` runs there, through the queue, with the project's
  lanes and estimates, using that checkout's own Makefile, ops script,
  Taskfile, or justfile; `skram discover` lists that checkout's targets.
  `-C <dir>` (`--checkout`) names a checkout explicitly and always wins. A
  directory outside the project's repo is refused with a message that says
  why.
- The `--json` output of `run`, `status`, `queue list`, `wait`, `logs`,
  `explain`, and `discover` names the checkout as `checkout` (absent when
  the job ran in the configured `path:`). Output for people shows it as
  `my-app/e2e (my-app--feature)` in `status`, `queue list`, `logs -F`,
  `explain`, `skram tui`, and the dashboard.
- A checkout removed while its job waited in the queue fails that job with
  the reason `checkout_removed`, and `skram explain` says how to re-run it.
- The MCP `run` tool takes an optional `checkout`, since the server cannot
  see which directory the agent works in.
- `skram agent install-rules` and `skram doctor` also cover each checkout's
  linked worktrees, so a new worktree gets its agent files.
- herdr: a job shows in the sidebar of the pane standing in its checkout, and
  the sidebar names the checkout.
- `skram workflow run` follows the checkout you are in: the steps whose
  project lives in that repo run there, from that checkout's own Makefile or
  script, and the other steps run in their configured `path:`. `-C <dir>`
  names the checkout explicitly. `skram explain` on a workflow now suggests
  a `skram workflow run …` command to re-run it.
- The MCP server sees config edits without a restart: a project added to or
  removed from the config shows in `discover`, `run`, and
  `skram://projects/<name>` on the next call.
- A job label no longer shows internal flags such as `--foreground` for a
  job with no project or target.

## [0.15.0] — 2026-09-22

- `skram kill` makes `skram wait` and `skram run -f` exit 137 and report the
  job as killed. Before this they reported the child's own exit code
  (often 1) and `failed`.
- `skram queue drain` with no processor running now says there is nothing to
  drain and does nothing. Before this the flag stayed set and stopped the
  next job's processor as soon as it started.
- `skram status` clears jobs left marked running by a processor that died,
  and text `status` shows when the queue is draining.
- `skram estimate` accepts `--config`, `--context`, and `--json` before or
  after the verb, and can take the project from `--context` or the current
  directory like `skram run`; a target's own flags still shape the estimate
  key. Before this, any flag before the project (such as `--config`) was
  ignored. `--json` prints the estimate as one object.
- A project's `guard.enabled: false` now turns the guard off for that
  project. Before this the key was read by nothing.
- `expose:` lists targets in the order written, everywhere they are shown.
- `skram doctor` exits 1 when it finds a problem (any `[!!]` line) and 0
  when every check passes. Before this it always exited 0; a script that ran
  `skram doctor` and expected success on a machine with problems now sees 1.
- `skram dashboard --local` prints `http://127.0.0.1:<port>`, the address it
  actually listens on, and no QR code. Before this it printed this machine's
  network address, which the dashboard never answered on. `--local` is for a
  browser on the same machine.
- `skram doctor` stops advising `skram init` for a data directory that does
  not exist yet; the directories are created on first use and the line is
  informational.
- `skram workflow run` queues the workflow as one job. The item names every
  lane its steps' projects declare, waits for all of them, and holds all of
  them until its last step ends, so a workflow step can no longer run beside
  a queued job on the same resource. It obeys a hold, names who asked, and
  takes `-f`, `--json`, and `--on-error` like `skram run`. Before this, a
  workflow ran its steps in your terminal outside the queue.

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
