# Changelog

What changed in each skram release, for someone running the binary. Versions
follow semver.

## [Unreleased]

## [0.24.0] — 2026-11-27

Jobs can read secrets by name, a target can name the inputs it reads, and a
resource can carry values from one job to the next. A project can show its
Grafana dashboards. The agent rules are now short, and jobs start and finish
sooner.

What changes on upgrade:

- `secrets:` and `--secret` need a `skram-vault` with `secret env` on the
  `PATH` of the machine that queues the job. Without one, a job that names a
  secret fails at once as `missing_secret`; jobs that name none are unaffected.
- A runner that is asked to take secrets needs this release too; an older one
  is refused as `runner_refused`.
- `skram agent install-rules` now also runs `skram-vault agent install-rules`,
  `skram-tunnel agent install-rules` and `skram-chat agent install-rules`,
  each when it is on `PATH` (an absent one is skipped with a note). Run it to
  refresh a checkout's rules, which are now at most 60 lines.
- Naming `docker` in a project's `tools:` is what makes Skram check the Docker
  engine before each job and start a stopped one; `skram explain` and
  `skram doctor` now say so when Docker fails in a project that does not.
- Setting `SKRAM_SSH` turns off the shared SSH connection to runners.
- Nothing else needs a config edit: a target you do not list under
  `target_inputs:` reads every input, as before.

### New

- **Jobs can read secrets by name.** A project's `secrets: [namespace/NAME, …]`
  and `skram run --secret namespace/NAME` (MCP `run`: `secrets`) are resolved
  by `skram-vault`, which Skram finds on its own `PATH`, and exported to the
  target under `NAME`. A job sent to a runner has its secrets resolved on the
  machine that queues it; a workflow step gets its project's secrets like any
  job. Skram keeps only the name: the value is blanked from the job's output,
  status and ledger as `‹secret:NAME›`, and a PEM key's `-----BEGIN-----` and
  `-----END-----` lines are left as they are. A secret that does not resolve,
  resolves empty, or takes longer than two minutes fails the job at once as
  `missing_secret`, an environment reason that leaves a ledgered target's
  streak alone. A secret may not be named like a variable Skram sets itself
  (`PATH`, `HOME`, `SKRAM_*`, `HERDR_*`). An ephemeral target runs without a
  job, so it gets no secrets and `--secret` on one is refused. The `secrets:`
  key is in the configuration reference.

- **A runner job gets its secrets over SSH, and nothing is written there.**
  They travel inside the SSH session and live only in memory on the runner
  (see "Secrets on a runner" in the runners guide). A runner whose Skram is too
  old to take them is refused as `runner_refused`, with a hint to upgrade it.

- **`--env` warns when a value looks like a secret.** A value `skram-vault` has
  handed out, or one shaped like a key or token, still queues the job but
  prints a warning on stderr naming the variable and pointing to `--secret`
  (`warnings` in `--json` and MCP `run`). `skram explain` re-run lines repeat
  the job's `--secret` names beside its `--env` flags.

- **`target_inputs:` says which inputs a target reads.** A project can list,
  per target, the input variables that target uses. Its jobs then report, key
  the ledger on, and check drift for only those; the others are still exported
  at their default checkout, and a missing default no longer stops the run. A
  target you do not list reads every input, as before. See `docs/config.md`.

- **Lane values.** A project that sets `lane_values: <resource>` gives its jobs
  a file named by `$SKRAM_LANE_ENV`. A target writes `KEY=VALUE` lines there
  (a key it just minted, say) and Skram keeps them for that resource: every
  later job on it, from any project or checkout on this machine, starts with
  those variables set. A later write of the same key replaces it, `KEY=`
  removes it, and a job that writes nothing changes nothing. Values are
  blanked in the logs of the jobs that receive them, and `status` and
  `explain` show names and which job wrote them, never values. A job sent to a runner
  takes part too: what it writes is kept for that resource on that runner,
  and later jobs sent there start with it, passed in memory and never written
  to the runner's disk. A job here does not see a runner's values, nor the
  reverse.

  `skram discover` (and `--json`) shows each listed target's inputs, the
  target tables `skram agent install-rules` writes gain an Inputs column for a
  project that uses `target_inputs:`, and `skram doctor` warns about a
  `target_inputs:` key that names no target.

- **A job on a runner receives only the inputs its target reads.** With
  `target_inputs:`, the other inputs' repos are not sent to the runner and their
  variables are not set there.

- **One command refreshes every tool's agent rules.** After its own section,
  `skram agent install-rules` runs `skram-vault agent install-rules` and
  `skram-tunnel agent install-rules`, then `skram-chat agent install-rules`,
  each when it is on `PATH` (a missing one is a one-line note), with your `--repos`, `--here`, `--check` and
  `--config`; `--projects` narrows Skram only. Each tool's output has its own
  heading, and the command exits non-zero when any of them does, after all have
  run. `--check --json` prints one document, with a `siblings` list and an `ok`
  that covers all four. Use releases of those tools that exit 0 where they
  have nothing of their own to write: with one that does not, this command
  fails there. See the agent setup guide.

- **Project dashboards in Grafana.** A project can set
  `metrics.dashboards: <folder>` (relative to its `path:`). `skram metrics up`
  mounts the folder read-only into the host Grafana as a folder named after the
  project and prints a line per project mounted; every dashboard file under it,
  at any depth, appears in that one Grafana folder. Edits to the files show up
  in about 30 seconds. A folder that is missing, is a file, or leaves the
  project's path is a warning, not a failure. `skram doctor` reports each
  declared folder as found, or as an issue. The metrics docs now name the ports
  (9090, 3100) and datasource uids (`prometheus`, `loki`) such dashboards and
  pushes rely on. See `docs/config.md`.

- **A Docker failure says how to get the engine started.** When a job fails because Docker stopped answering and its project does not name `docker` in `tools:`, `skram explain` names that line, which makes skram check the engine before each job and start a stopped Docker Desktop or OrbStack once. `skram doctor` says the same when the engine is down and no project that runs on this machine declares it. See `docs/config.md`.

### Changed

- **The agent rules are short, and the rest is read on demand.** What
  `skram agent install-rules` writes into a checkout (and the MCP server
  serves as `skram://guide`) is now at most 60 lines instead of about 230. The
  rules about the ledger, failure reasons, runners, job files and configuration
  moved to topics: `skram rules` lists them, `skram rules <topic>` prints one,
  and the MCP server serves each as `skram://rules/<topic>`. Run
  `skram agent install-rules` to refresh a checkout. `skram explain` now tells
  an agent, for every failure caused by the machine (`disk_full`,
  `docker_down`, `missing_tool`, `missing_secret`, `setup_failed`,
  `start_failed`), not to enqueue the job again and to report the machine to
  you.

- **JSON reports show the job's command as you would type it.** `status`,
  `queue list`, `run`, the dashboard, and the MCP tools no longer put the
  queue's own flags (`--foreground`, `--_job-id`, a leading `--config <file>`)
  in a `command` field. `explain --json` gains `command` and `rerun`: the
  job's line, and the full line to run it again with its `--env` and
  `--secret` flags, so a caller no longer strips a prefix off a `next` hint.

- **Short jobs return sooner.** A job queued while another lane is busy starts
  at once instead of up to 5 s later, and `run -f` and `wait` notice a job's
  start and end within tens of milliseconds instead of half a second. A
  trivial `skram run <project> <target> -f` takes about 0.1 s instead of 0.5 s.

- **Runner jobs start faster.** Skram's calls to a runner now share one SSH
  connection (`ControlMaster`, kept 60 s), so a short job on a runner spends
  about a second less on logins. Setting `SKRAM_SSH` turns this off: your
  program is run exactly as before.

### Fixed

- **Secrets follow `--config`.** Running skram with `--config X` now makes it hand the same file to `skram-vault` when it resolves a job's secrets, so a job under a non-default config no longer fails `missing_secret` with "config file not found".

- **`skram tui` runs a scoped target.** Launch now works out each target's
  inputs on its own, so a target that does not read an input with no checkout
  here is no longer refused, and standing in a checkout of an input a target
  does not read no longer exports it.

- **A target defined only in a worktree is scoped by that worktree.**
  `target_inputs:` looked the target up in the configured checkout's file, so a
  worktree-only target read every input; the CLI and MCP now agree.

- **A skipped or in-flight run shows the command you would type.** The
  `--json` report no longer carries the internal `--foreground` flag.

- **A job lost before it came home still leaves its lane values.** They are
  recorded when the runner is next reached, and no copy stays on disk.

- **A `-C` refused for an input the target does not read** now says so in one
  sentence and names the inputs the target does read.

- **`skram agent install-rules` gets its exit code right.** A run whose
  checkouts all have no projects now succeeds instead of failing with "no
  artifacts were installed", and a file that cannot be written or removed now
  makes the command exit non-zero even when other files were written.
  `--here` now works in every checkout a plain run visits, including a linked
  worktree of a `repos:` entry that has only `paths:`, no projects, and no
  origin; outside git it says the directory is not inside a git checkout.

- **A shallow clone can be sent to a runner.** A job reading a repo with a
  shallow submodule (or a shallow checkout or input) failed before it started
  with git's `shallow update not allowed`. It now runs, with nothing to do on
  the runner first. A `git push` the runner rejects is reported as `runner_refused` with
  git's message, no longer as `runner_unreachable`.

- **An artifacts archive holds only the files its job wrote.** A file the
  project's `artifacts:` globs match but that was last modified before the
  job started (a report an earlier job left in the checkout) is left out, so
  a job that writes no report no longer packs the last one again.

- `skram explain` re-run lines now repeat the job's `--env` flags, so pasting
  one reruns the job with the same environment instead of silently dropping it.

### Documentation

- **The record leads the docs.** The README, the agent setup page, the MCP tool descriptions and the rules say what each job keeps (the commit each checkout and input was on) and that `explain` on a green job gives its env, checkouts and `rerun` line. A new recipe covers picking up where the last session stopped.

## [0.23.0] — 2026-11-24

Runners get the checkout's submodules, a setup step, a phase you can watch,
and storage that prunes itself, and every job now checks the machine it starts
on (free disk, the tools it needs, the Docker engine), reaping or starting the
engine once before it fails a job for them.

What changes on upgrade, with no config edit:

- Every job checks free disk first and fails as `disk_full` below 5GB
  (`preflight.min_free_disk`; `0` turns it off).
- A project whose `tools:` names `docker` fails as `docker_down` when its
  engine does not answer, and as `disk_full` below 10GB inside the engine
  (`preflight.docker_min_free`).
- `skram runner check` and `skram doctor` exit 1 on a short disk or a down
  engine.
- A runner removes worktrees idle longer than 72h, ignored files included
  (`prune_after:`).
- `skram runner add` and `skram init --detect` exit 1 when nothing was written
  for want of a terminal and `--yes`.
- A failure's `reason` and `reason_detail` may differ for the same output, and
  five reasons nothing ever set are gone.
- The MCP `wait` tool returns after at most 25 minutes.

### New

- A job sent to a runner gets the checkout's git submodules, nested ones too,
  at the commit the checkout records, pushed from this machine; the runner
  needs no access to their remotes. A submodule with uncommitted changes ships
  them and marks the job's `sent` dirty, so a ledgered target records nothing;
  one you have not initialised is absent there, with a `missing_submodule`
  warning. `sent` lists them under `submodules`. The manual `git submodule
  update` on the runner is no longer needed.
- `setup:` on a project names a target that runs first, as its own step, in
  every job sent to a runner and every local job in a worktree or another
  clone; jobs in the project's `path:` skip it. A failing setup ends the job
  as `setup_failed` before the target starts, and `skram explain` says so.
  `skram discover --json` shows it as `setup`.
- `skram status` shows what a running job is doing, its phase, on its line,
  and `status --json` and `queue list --json` carry it as `phase`. A job sent
  to a runner goes `checking`, `shipping`, `preparing`, `running`, `homing`,
  so a large push no longer looks like a hang; a local job is `running`.
  Either is in `setup` while the project's setup target runs.
- A runner's disk stops growing: each job sent to it removes the worktrees
  nobody used and the job files that came home longer ago than the runner's
  `prune_after:` (default `72h`, `0` keeps everything). **Behaviour change:**
  a runner worktree's ignored files (installs, caches, anything copied in by
  hand) are gone after it sits idle that long; the setup target rebuilds an
  install on the next job.
- A project's `tools:` names the commands its targets need. Before a job
  starts, each is looked up on the `PATH` of the machine it will start on,
  this one or a runner's login shell; a missing one fails the job at once with
  reason `missing_tool`, naming the tool and the machine, and nothing is sent
  to the runner. `discover --json` shows a project's `tools`, and `explain`
  says what to fix. A runner's own `tools:` still only warns.
- A project's `artifacts:` globs (with `artifacts_limit`, default `500MB`)
  name files a job makes, such as test traces, reports and logs. When a job
  ends, passed, failed or killed, the matches are packed into
  `artifacts.tar.gz` in its job directory; a job sent to a runner brings its
  archive home. `wait --json` and `explain --json` name it as `artifacts`,
  `status --json` as `most_recent_artifacts`, and `explain` prints it. The
  globs are relative to the project's directory in the checkout.
- `skram runner list` shows what the last contact with each runner found,
  without contacting it: `LAST PROBE` (`ok`, `unreachable`, `declined` or
  `never`, and how long ago), `FREE DISK`, and `DOCKER` (the engine its
  `docker` context reached, e.g. `OrbStack (context orbstack), up, 31GB
  free`); `-` is a value that was not measured. `--json` carries the same
  fields as `skram status --json`'s `runners`, which now also documents
  `host_free_bytes`. `skram runner check` gains a `disk` row and, for a runner
  whose projects use Docker, a `docker` row; `skram doctor` gains this
  machine's disk and Docker lines, and each runner's line shows its disk and
  engine.
- `skram reap --runner <name>` reaps a runner's Docker now, with this
  machine's `reaper:` section and the command's flags; the output streams back
  and the runner needs no config. A job sent to a runner that matches
  `reaper.after_targets` now reaps on the runner afterwards, inside the job,
  as a local job does. On an engine using the containerd image store, a reap
  logs one line explaining why a build cache prune can report 0B.
- `reaper.tag_patterns` takes several tag patterns, each keeping its own
  newest `keep_tags` per repository, and `reaper.keep` lists `repository:tag`
  globs that are never removed. Newest now means the image's creation time,
  not the tag text. `tag_pattern` still works as a one-item list; setting both
  keys is an error.
- On macOS a running job holds its machine awake for exactly its length
  (`caffeinate`, bound to the job's process), on this machine and on a runner,
  and this machine stays awake while it streams a runner job. Pending and
  waiting items hold nothing. `skram runner check` now reports a runner that
  allows sleep as information, with no fix to apply.
- The TUI's Launch tab has a runner picker: `ctrl+r` cycles `local` and each
  declared runner, starting on the project's default, and the footer shows
  where the job will wait. The Queue tab says why a pending item waits (for a
  runner, since when and when the next try is; for a lease, who holds it and
  when it should free), and a running job's row in Jobs shows its phase, in
  the same words as `skram status`.

### Changed

- **On by default:** before a job starts, Skram checks free disk on the
  machine it will run on, this one or a runner, against the new
  `preflight.min_free_disk` (default `5GB`; `0` turns it off). A job on a
  machine below the floor fails at once with reason `disk_full`, naming the
  machine, the free space and the floor, instead of starting; a runner job
  fails before anything is sent. Ephemeral targets are not checked. `skram
  explain` says what to free.
- **On by default:** a job of a project whose `tools:` names `docker` first
  checks the Docker engine its `docker` context reaches on the machine it will
  run on (Docker Desktop, OrbStack, or a plain daemon). An engine that does
  not answer within 30 seconds fails the job at once with the new reason
  `docker_down`; less free space inside the engine's own storage than the new
  `preflight.docker_min_free` (default `10GB`; `0` turns it off) fails it with
  `disk_full`, the detail naming the engine and context. A runner's engine is
  checked before anything is sent to it, and `skram status --json` shows what
  the last check found.
- When a job's preflight finds a disk floor short and the `reaper:` block is
  on, Skram reaps once on that machine (a runner's too, before anything is
  sent to it) and measures again before failing the job. The reap is a step of
  its own in the job's log and is never queued. With room after it, the job
  runs; still short, it fails `disk_full` and `reason_detail` says a reap was
  tried and what it freed. A `docker_down` engine is never reaped.
  `SKRAM_REAP_DISABLE=1` turns this reap off too.
- When a job's Docker engine does not answer, Skram starts it once on that
  machine (`docker desktop start` for Docker Desktop, `orbctl start` for
  OrbStack; a runner's too, before anything is sent) and waits up to
  `preflight.docker_start_timeout` (default 2m) for it to answer before
  failing the job as `docker_down`. The attempt is a step of its own in the
  job's log and is never queued; a failed one is named in `reason_detail`.
  `preflight.start_docker: false` turns it off.
- `skram runner check` and `skram doctor` now fail (exit 1) when a machine is
  below `preflight.min_free_disk`, or its Docker engine is down or below
  `preflight.docker_min_free`, as a job sent there would. `doctor` shows this
  machine's Docker engine whenever a `docker` program is on `PATH`; it is a
  problem only when a project here uses Docker.
- `skram runner check` also looks for the `tools:` of every project whose
  default runner it is; `--tools` still replaces the list.
- On a target in a project's `ledger:`, a job that fails because the machine
  was unfit (`disk_full`, `docker_down`, `missing_tool`, `setup_failed`,
  `start_failed`) is no longer recorded as a red: the key's streak and flakes
  stay as they were, so `--need-greens` keeps its count and a sound branch is
  never marked flaky by a full disk. `skram explain`'s `ledger` section says
  nothing was recorded. Fix the machine, then run the same line again.
- A failed job's `reason` and `reason_detail` are more precise, and may differ
  from before for the same output. Rules now match whole words or a line's
  start, so a log that only mentions "expected", "lint" or "tsc" is no longer
  a test, lint or compile failure. `reason_detail` is the line that matched
  (for a red test suite, the test runner's summary line: Playwright, Jest,
  Vitest, pytest, or Go's package `FAIL` line) instead of whatever the log
  printed last. A full disk mid-job (`no space left on device`) is
  `disk_full`, and a Docker engine that stopped answering is the new
  `docker_down`; both win over anything printed after them, and `skram
  explain` says how to recover. A missing command or module is `script_error`.
  The reasons `timeout`, `user_cancel`, `code_bug`, `cve` and `act`, which
  nothing ever set, are gone. The list of reasons is in the MCP docs under
  `explain`.
- A job that ends before its target runs (its process cannot start, the config
  no longer loads, the checkout or backend file is gone, or the target's tool
  is not on `PATH`) now fails with the reason `start_failed` and the error as
  its detail, its log ends like any job's, and `skram explain` names the cause
  and suggests `skram doctor`. Such jobs used to show no reason.
- A job's estimate prefers the history of the machine it runs on. A job sent
  to a runner uses that runner's past runs when it has any (a job run here,
  the local ones), and all runs otherwise, so a slow runner's job no longer
  reads as overdue and `wait --timeout auto` gives it enough time. `skram
  estimate --runner NAME` asks for a runner and says whose runs it used.
- The MCP `wait` tool now defaults to and is capped at 25 minutes (was 30
  minutes, max an hour), so it returns before an MCP client gives up on a
  silent tool. A job still going comes back as `status: timeout` with
  `still_running`, `phase` and `elapsed_seconds`: call `wait` again with the
  same ref. A client that sends a progress token gets a progress notification
  every 30 seconds. `skram wait` is unchanged.
- `skram runner add` and `skram init --detect` now exit 1 when nothing was
  written because there was no terminal and no `--yes`. The proposal still
  prints, and "Nothing written" goes to stderr. `--dry-run`, `--json`, and
  answering `n` at the prompt still exit 0.
- In `skram tui`, the Jobs and Queue tabs show where a job runs in a column of
  its own (`local` when it runs here) instead of an `@name` after the job, and
  their columns stay aligned with colour on and long or wide names.
- When a runner job dies and its leftover process groups are killed, `skram
  explain` on the queue machine names them.

### Fixed

- A job on a runner whose process died without finishing (killed with SIGKILL,
  out of memory, a crash) left its target's processes running there, working
  on a resource the next job was already using. The runner now kills them when
  it finds the job dead.
- Removing the directory a `skram run` started from (a worktree, say) no
  longer makes every later job fail at once with an empty log. The queue's
  background worker and its jobs now start in Skram's queue directory; a job
  still fails as `checkout_removed` when its own checkout is gone.
- Uncommitted changes inside a git submodule, or a submodule Skram cannot
  read, now count as a dirty checkout for a `ledger:` target, so that run is
  no longer skipped as already green and its green is not recorded.
- A workflow's project setup runs before the project's first step that needs
  it, even when its first step is an ephemeral target.
- `runner check` and `doctor` now report git, the login `PATH` and every tool
  on a runner running an older Skram, instead of "not probed".
- A script arm such as `help|--help|-h)` now gives a `help` target; before,
  the whole arm was ignored because of its flag alternatives.
- `skram tui` in a short terminal keeps the tab bar, the running jobs and the
  footer; history rows go first, then the log pane, and below 3 rows it shows
  a one-line notice.

### Docs

- A workflow job does not pack artifacts or run the `tools:` and Docker engine
  checks; the config page says so.

## [0.22.1] — 2026-11-20

- **New:** install with Homebrew on macOS: `brew trust skramstudios/tap`, then
  `brew install skramstudios/tap/skram`. The cask installs the shell
  completions and clears the quarantine flag; `install.sh` still works.

## [0.22.0] — 2026-11-19

- **New:** an `e2e:` block on a project in your config (`scopes`, `iterate`,
  `fast`, `deploy`, `gate`, `dev`, `runners`) holds that project's end-to-end
  commands. `skram agent install-rules` writes it into `AGENTS.local.md` as an
  `### E2E recipe` table, and says so when a project has a `test` or
  `cluster-run` target but no `e2e:` block.
- **New:** `ledger:` in a project lists its proof targets. A job of one that
  exits 0 on committed trees is recorded green: the git tree of its checkout
  and of each input, its target, and its arguments. Asking for the same run
  again does not queue it: `skram run` exits 0 and names the green job and
  how long ago it passed (`skipped: true` in `--json` and from the MCP `run`
  tool), and `skram explain` on that job says why and how long ago the key's
  newest green passed. The arguments are kept in the order given, so
  `--grep A --project B` and `--project A --grep B` are different keys; only
  skram's own flags are left out. `skram wait <job>` and `skram run -f`
  return only once the job's outcome is recorded. `--force` (MCP `force`)
  runs it anyway. A job sent to a runner is recorded on the machine that
  holds the queue. A project without the list, and every target not on it,
  runs every time, as before.
- **New:** `skram run --need-greens=N` (MCP `need_greens`) on a `ledger:`
  target runs until the same trees, target, and args have N consecutive
  greens, then skips: each `run` queues at most one job, and asking again
  once the count is met adds nothing. Jobs of the same key still pending or
  running count too: once the greens plus those reach the count, nothing is
  queued and `run` exits 0 with `in_flight: true` and `in_flight_items`, each
  with a `wait` command (`run -f` follows the oldest instead), so a gate line
  sent again never adds a job. A failing job is recorded red and resets the
  count, so a plain run after a red runs again. A red right after a green is
  recorded as a flake (a red after a red is not): the skip report counts
  them (`flakes`) and `skram explain` lists them. Every argument, `--fast`
  included, stays part of the key.
- **New:** a ledgered job can report the trees it actually tested, such as
  the images on a cluster, by writing `<label> <tree-hash>[ dirty]` lines to
  the file named in `SKRAM_TESTED_TREES`. The lines replace the checkout's
  and inputs' trees in the key. A report that names only trees the job
  started on also lets the next identical run skip; one that names older
  trees never skips a run of the checkout. A `dirty` or malformed line
  records nothing.

## [0.21.0] — 2026-11-18

- **New:** `skram runner`, three commands for the machines you send jobs to.
  `skram runner list` shows each declared runner with its SSH destination,
  its tools, and the projects that default to it. `skram runner check
  [name...] [--tools a,b]` checks a machine from here, one row each: SSH
  without a prompt, the same OS and architecture, the Skram build there,
  that it accepts jobs, `git` on the plain SSH `PATH`, the login-shell
  `PATH`, each tool your jobs need on it, and, on a Mac, that it does not
  sleep. The fix follows every failed row, it exits 1 on a failure, and it
  sends and pushes nothing. `skram runner add <name> [--ssh <dest>]
  [--tools a,b]` writes the entry into your config the way `skram init
  --detect` writes a project (it asks in a terminal, writes elsewhere only
  with `--yes`, and `--dry-run` or `--json` write nothing), then runs the
  check. All three take `--json`.
- A runner may list the commands its jobs need under `tools:`. `skram
  doctor` and `skram runner check` report any that are missing from the
  runner's login-shell `PATH`, and say what that `PATH` is. A job sent to a
  runner that lacks one still runs, with a warning in its log.
- `skram doctor` ends each runner line with when that runner last answered,
  and says so when no runner is declared. `skram status --json` and `skram
  discover --json` carry a `runners` object: the first the last answer of
  each runner, read from a file with no SSH call, the second each runner's
  destination and tools.
- `skram explain` on a runner job that failed with `command not found` says
  the job's `PATH` came from the runner's login shell and shows the `--env
  PATH=...` re-run that overrides it.
- The agent rules have a Runners section: what a runner is, how to list
  them, that nothing spreads jobs by itself, and that an agent never logs in
  to a runner or installs anything there. `skram agent install-rules`
  carries it into each checkout.
- The generated config shows a commented `runners:` example with `tools:`.
- `docs/runners.md` has a checklist for setting up a machine as a runner
  with a command to check each requirement by hand, what is not true about
  runners, and an example that splits a suite across the local queue and
  two runners. `docs/troubleshooting.md` has rows for a Docker login that
  fails over SSH (`-25308`), a sleeping runner, and git submodules, which
  are not initialised in a runner's worktree.

## [0.20.0] — 2026-11-09

- **Fixed:** the guard blocked commands that never touch the queue: another
  repo's own `scripts/ops.sh` (every ops script matched by name, from any
  directory), a syntax check (`bash -n`), and text that only mentions a
  command (a heredoc, a quoted commit message). It now parses the command
  as a shell would: `cd` moves where the rest is judged, a bare script name
  is guarded only inside its own checkout, quoted text and heredocs are
  data unless a shell runs them (`bash -c`, `eval`, `bash <<EOF`), and
  `-n` runs nothing. See `docs/agent-setup.md`. The agent rules now tell an
  agent to report a block it believes is wrong instead of working around it.

- **Fixed:** Prometheus could not scrape a runner whose `ssh:` host is an
  alias in your SSH config or a name only this machine resolves (a Tailscale
  or `.local` name). `skram metrics up` now resolves each runner's host here
  and writes its address into the scrape targets.
- `skram metrics up --exporter-only` starts the exporter alone, for a runner
  whose metrics another machine's Prometheus scrapes. On a machine with no
  compose file, `skram metrics down` and `skram metrics status` now stop and
  report the exporter and say there is no stack, instead of failing.

## [0.19.0] — 2026-09-30

This release sends jobs to other machines. The queue stays on the machine
you work on; a job can run on any machine you reach over SSH, and each
resource is still used by one job at a time, wherever the jobs come from.
See `docs/runners.md`.

- **Breaking:** `skram metrics up` runs the Prometheus exporter as a
  process on this machine instead of in the compose stack, listening on
  port 2112 (`--addr` changes it); `skram metrics down` and `skram metrics
  status` stop and report it with the rest of the stack. Before starting
  compose, `metrics up` writes a Prometheus file_sd targets file naming this
  machine and every runner, each labelled `machine`, and passes its
  directory as `SKRAM_METRICS_TARGETS_DIR`, so one Prometheus can scrape
  every machine that runs `skram metrics up`. A compose file written for an
  earlier release must drop its exporter service and mount that directory
  into Prometheus; see "The metrics stack" in `docs/how-it-works.md`.
- A job can run on another machine, a **runner**. Name the machines you
  reach over SSH under `runners:` in the config, then `skram run --runner
  build-box api test` queues the job here as usual and, when its turn
  comes, runs it there. `wait`, `logs -F`, `run -f`, `explain`, `status`,
  and `kill` work as they do for any job: the output streams back as it
  prints, and the job's `raw.log`, `events.jsonl`, and `status.json` are
  copied home when it ends. A project can name a default with `runner:`,
  `--runner local` runs one job here instead, and the MCP `run` tool takes
  `runner`. Labels show where a job ran (`api/test @build-box`). Skram copies
  itself to the runner, which needs only `git`, your tools, an SSH login
  without a prompt, and the same OS and CPU architecture as this machine.
  The job's `PATH` is the runner's login shell's, so what `~/.zprofile`
  adds there reaches it.
- What a runner receives is the commit you are on, plus your uncommitted
  edits and any files git does not ignore that you have not added yet. Files
  git ignores, such as `node_modules`, build output, and `.env` files, are
  never sent, and nothing changes in your branches, tags, remotes, or index.
  The runner keeps one working copy per checkout, so ignored caches survive
  between jobs there. A project with `inputs:` sends each input's checkout
  the same way, and `sent` in the job's `--json` output names the exact
  commit that ran. The project's `env:` and `--env` travel with the job.
- A job sent to a runner has its own lane, `<resource>@<runner>`
  (`cluster@studio`): a local job and a job sent to another machine on the
  same resource run at once, and two jobs sent to one runner take turns.
  `skram status`, `skram queue list`, `skram run`, `skram explain`, and the
  TUI name the lane (`lanes` in `--json`, beside `resources`, which stays
  the resource names), and `skram wait --resource cluster@studio` and
  `skram kill --resource cluster@studio` take it. `skram queue list` now
  shows a queued or running job's lanes after its label (`[cluster]`)
  whenever they are not just its project's name.
- On the machine that has a resource, a job sent there and a job queued
  there never run at once: each resource has a lease on that machine, taken
  by its own jobs and by jobs sent to it alike. When a job you send finds
  the resource busy there, nothing starts and nothing fails: the item waits
  at the head of its lane and is sent once the resource is free, and `skram
  status`, `skram queue list`, and `skram wait` (`waiting_for_lease` in
  `--json`) say which job holds it, from which machine, since when, and
  when it should be done. The runner's own queued job waits the same way
  and starts by itself, and `skram status` there lists the leases held by
  jobs sent from another machine (`leases` in `--json` has every one). A lease whose job died, or one
  taken before the machine last booted, is taken over by the next job that
  needs the resource.
- A runner that is off does not fail your job: the item waits at the head
  of its lane (`waiting_for_runner`), Skram asks again about once a minute,
  and `skram status` shows the wait; `skram queue cancel` withdraws it as
  usual. A connection that drops while a job runs is reattached (at once,
  then after 15 and 45 seconds) with no line of the log lost or repeated;
  if it cannot be, the job fails as `runner_lost`, and the next time Skram
  reaches that runner it stops the job there and copies its files home. A
  runner that answers but cannot take the job fails it as
  `runner_unreachable` or `runner_refused`, and `skram explain` says why.
- A machine's owner can set `accept_runner_jobs: false` in their own config
  file, and every job any machine sends it fails at once with reason
  `runner_declined`, with nothing run there; `skram explain` names the
  runner and the setting. A job already running there finishes and comes
  home. Removing the key, or setting it `true`, makes the machine a runner
  again. `docs/runners.md` also notes how Tailscale SSH fits: it is off per
  device until turned on, and the plan limits.
- `skram doctor` reports each runner: whether it answers, its OS and
  architecture, its Skram build against this one, and whether it declines
  jobs.
- The exporter reads the queue and the leases as well as finished jobs:
  `skram_queue_wait_seconds` is how long each job waited from enqueue to
  start, per project, lane, and runner; `skram_lane_busy` and
  `skram_queue_pending_items` say whether each lane is in use and how many
  items wait in it; `skram_lease_held` is 1 for each resource whose lease a
  job holds, labelled with the machine that job came from (`holder_machine`)
  and whether it was sent from another machine (`inbound`).
  `skram_jobs_total` and `skram_job_duration_seconds` gain a `runner` label
  (`local` for a job that ran here). Every job's `status.json` now records
  `queued_at`, so the wait is known after the queue archives the item.
  `docs/how-it-works.md` lists every metric with example queries.
- Fixed: the exporter labelled queued jobs with project `--foreground`
  instead of their project and target.
- Fixed: `skram status` and `skram run` printed `queue clears in ≥<1s` when
  nothing queued had run before; they now say `no queue estimate yet`.
- The README's recording shows a run enqueued from a worktree landing in the
  same lane, and the README names checkout runs, `-C`, `inputs:`, and
  `--env`.

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
