# Changelog

What changed in each skram release, for someone running the binary. Versions
follow semver.

## [Unreleased]

## [0.18.0] — 2026-09-25

- **Breaking:** a job that was killed is now recorded as `killed` with exit
  code 137 everywhere. Before, `skram status` and `skram queue list` showed
  it as `failed` with exit code `-1` while `skram explain` and `skram wait`
  said `killed`. If you read the JSON, `skram status --json`
  (`most_recent`), `skram queue list --json`, `skram explain --json`, and
  `skram wait --json` now report `"status": "killed"` and `"exit_code": 137`
  for a killed job; `skram status --json` has a `killed` count beside
  `failed`, and `skram queue list --json` a `killed` entry in `counts`. Jobs
  killed by an earlier release read the same way.
- A job ended by any signal, not only `skram kill` — the out-of-memory
  killer, a `kill` from another shell — is recorded as `killed` too, with
  `reason: signal` and the signal's name (such as `SIGKILL`) as
  `reason_detail`; `skram explain` and the completion notification name it.
  Before, it was recorded as `failed`, with exit code 1 or `-1`.
- Killing a queued job's process with `kill -KILL` (or a `skram kill` that
  had to fall back to SIGKILL) no longer leaves the command it was running
  behind. Before, the target script and everything it started kept running
  after the job was recorded as `killed`, holding whatever it was using
  while the next job in its lane started. Now skram stops them before it
  records the job, which still reads `killed` with exit code 137.
- `skram kill` on a job that has already finished refuses and says how it
  ended, instead of acting on it.
- **Breaking:** a job now cleans up everything its command started even when
  the command exits normally on its own, not only when it's killed. Before,
  a command that started a background process and then exited could leave
  that process running after the job was recorded as done, holding whatever
  it was using while the next job in its lane started. If your command
  relies on something outliving the job on purpose, give it its own session
  (or run it under a service manager) — the job's log now names anything it
  had to clean up this way, so it's clear it's gone and why.
- A project can name the other repos its targets read, under `inputs:`: one
  environment variable per repo, such as `LIB_DIR: my-lib`. A job follows
  each input the way it follows the project's own checkout: to the checkout
  of that repo you stand in, to one you name with `-C`, and otherwise to the
  repo's first `paths:` entry that exists on this machine. The target sees
  each variable set to that directory and `SKRAM_INPUT_<VAR>_DEFAULT` set to
  the default one. A run is refused before anything is queued when an input
  has no checkout on this machine and you named none. `skram discover` lists
  each project's inputs, `skram doctor` reports an input with no checkout
  here, and the targets table `skram agent install-rules` writes names each
  input's variable and repo. See `docs/config.md`.
- `-C <dir>` can be given more than once to `skram run`, `skram discover`,
  and `skram workflow run`, one directory per repo: the project's own
  checkout and one for each input. Two directories for one repo are refused.
  With no inputs, a single `-C` works as before.
- Every job records where its inputs resolved. The queue item, `status.json`,
  the `job_start` and `job_end` events, and the `--json` output of `run`,
  `status`, `queue list`, `wait`, `explain`, and `logs` carry `inputs`, keyed
  by variable, with the repo, the directory, its default, and the commit it
  was on when the job was queued and when it started. Labels name each input
  that is off its default after the checkout, as in
  `my-app/build (my-lib--x)`, in `status`, `queue list`, `logs -F`,
  `explain`, `skram tui`, the dashboard, and the herdr sidebar. Where an
  input resolved is fixed when the job is queued; editing `paths:` while it
  waits does not move it.
- An input whose commit moved between queueing and starting gets one warning
  line in the job log and a `warn` event tagged `input_drift`; the job still
  runs. An input whose directory is gone when the job's turn comes fails the
  job with `checkout_removed`, and `skram explain` suggests how to re-run
  it.
- The TUI's Launch tab follows inputs from the directory it was started in,
  like `skram run` does, and shows them in its preview.
- The MCP `run` tool takes `checkouts`, a list of directories, one per repo,
  for a project with inputs, and `env`, variables for that job only. Its
  result carries `inputs`.
- `skram run --env NAME=VALUE` sets a variable for that job alone, over the
  project's `env:` and your environment. It can be given more than once, and
  it may not set an input's variable; use `-C` for that.
- `--on-error continue` on a pending item now also keeps it from being
  cancelled when a job ahead of it in its lane fails under `stop`. Before
  this, a `stop` failure cancelled it anyway.
- `reaper.timeout` (default `5m`) bounds each destructive docker call the
  reaper makes and the whole reap that runs after a job. Read-only docker
  calls stop after 30s, so a Docker daemon that hangs can no longer hold up
  the job's lane. A `docker system df` that times out only skips the disk
  usage report.
- Fixed: a shell script's `case` command whose arm was written on a single
  line — for example `deploy) ./do-deploy.sh; exit 0 ;;` — was silently
  dropped from discovery, so running it said the command was not found.
  Arms written this way now show up like any other, at any indentation
  (including none), with a quoted name (`"deploy")`), and with more than one
  name on the arm (`a|b)`). `skram init --detect` also now recognizes a
  script whose only arms are written this way.
- `skram run -f` and `skram wait` could report a job done, with the wrong
  (too short) duration, while its post-job reap was still running and its
  lane was still held. Both now wait for the reap and report the same
  duration `skram status` does.
- Every command prints the whole job id, as in `J20260924_105251_84ef6a`.
  `skram status`, `skram explain`, `skram wait`, and notifications used to
  cut off the suffix, printing an id that `skram explain` then refused, and
  two jobs started in the same second looked the same.
- `skram explain`, `skram logs`, `skram wait`, and `skram kill` (and the MCP
  `explain`, `logs`, and `wait` tools) each take a job id or a queue item id
  (`q_…`), whether the item is still queued or already in the queue's
  history. `skram wait` on an item from history now answers with how it
  ended instead of refusing it. Ids must be given whole.
- `skram kill <item id>` kills that item's job. On an item that has not
  started it refuses and tells you to run `skram queue cancel <item id>`.
- `skram queue list` shows each item's id and, once it has started, its job
  id, so there is something to paste into `queue cancel`, `wait`, or
  `explain`.

## [0.17.0] — 2026-09-23

- `skram agent install-rules` removes a skram section an older skram wrote
  into `AGENTS.local.md` in a checkout that has no projects; before, it was
  left in place and `--check` passed over it. Anything else in the file,
  including skram-vault's section, stays as it is, and `CLAUDE.local.md`'s
  import goes only when the file itself is removed. `skram doctor` lists the
  leftover section until then.

- **Breaking:** `skram lore`, `spec`, `ticket`, `vault`, and `tunnel` no
  longer redirect. Each is now an unknown command; run `skram-vault` or
  `skram-tunnel` directly instead.

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
