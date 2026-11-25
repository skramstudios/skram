# The MCP server

`skram mcp` serves the queue to an editor or an agent over MCP on stdin and
stdout, until the client disconnects.

```bash
skram mcp
```

Its six tools return the same reports the `--json` flags print — one set of
builders, so a tool result and the matching command's `--json` output are the
same object. Nothing but the protocol is ever written to stdout.

[Agent setup](agent-setup.md) covers registering the server, the
machine-local files `skram agent install-rules` writes into a checkout, and
the guard hook. [How it works](how-it-works.md) is the behaviour behind these
reports, and the [configuration reference](config.md) is the file they read.

Every example below is real output from a machine with one project, `apps`,
whose targets are `build`, `deploy`, `test`, and an ephemeral `status`.

## What the client gets at connect time

`initialize` answers with the server's name and version and a block of
instructions. The instructions are the agent rules — the same text the
`skram://guide` resource serves and `install-rules` writes into a checkout —
followed by one line per project and a pointer to the resources:

```text
## Shared-resource jobs (skram)

… the rules: the loop, what to do when the queue is held, ephemeral
targets, the guard, where a job's files are …

Projects and targets (typical duration from history; [ephemeral] runs immediately):
- apps: build (build the image, ~1s) · deploy (roll the new image out, ~12s) · status (print what is running) [ephemeral] · test (run the test suite)

Read skram://projects/<name> for a project's full target table and skram://guide for the rules in detail.
```

Which projects are listed depends on the directory the client started the
server in. When the `repos:` block recognises that directory as a checkout,
only that checkout's own projects are listed, under a line naming the repo;
every other project is still reachable through `discover`. When it recognises
nothing, all of them are listed.

So an agent that reads the instructions knows the project names, the target
names, and roughly what each costs before it calls anything.

## The six tools

| Tool | The command it mirrors |
| --- | --- |
| `discover` | `skram discover --json` |
| `status` | `skram status --json` |
| `run` | `skram run <project> <target> --json` |
| `wait` | `skram wait <ref> --json` |
| `explain` | `skram explain <ref> --json` |
| `logs` | `skram logs <ref> --json` |

The loop is `discover` once, then `run` → `wait`, and `explain` when something
fails. Prefer `wait` to polling `status`.

### discover

Projects, their targets, the help text the backend file carries, whether a
target is ephemeral, and the typical duration from this machine's own history.
Call it first whenever a project or target name is not certain. It always
reads each project's configured `path:`, wherever the server was started;
`skram discover` in a shell follows the checkout it is run from.

Input, all optional:

```json
{ "project": "apps" }
```

Output:

```json
{
  "projects": {
    "apps": {
      "path": "/home/you/dev/apps",
      "backend": "script",
      "file": "/home/you/dev/apps/ops.sh",
      "resources": ["docker"],
      "script": "/home/you/dev/apps/ops.sh",
      "targets": [
        {
          "name": "build",
          "help": "build the image",
          "ephemeral": false,
          "estimate_seconds": 1,
          "estimate_samples": 3,
          "estimate_level": "exact"
        },
        {
          "name": "deploy",
          "help": "roll the new image out",
          "ephemeral": false,
          "estimate_seconds": 12,
          "estimate_samples": 1,
          "estimate_level": "exact"
        },
        { "name": "status", "help": "print what is running", "ephemeral": true },
        { "name": "test", "help": "run the test suite", "ephemeral": false }
      ]
    }
  },
  "runners": {
    "studio": { "destination": "me@studio.example.net", "tools": ["node", "pnpm"] }
  }
}
```

`runners` sits beside `projects`: every runner the config declares, keyed by
name, with its SSH `destination` and its `tools` (see
[Runners](runners.md#declaring-a-runner)), and `{}` when there is none. A
project's own `runner` is the one its jobs go to when a run names none, and
its `tools` (absent when it declares none) are the commands each of its jobs
needs on the machine it starts on (see [`tools:`](config.md#tools)).

`backend` is `script`, `make`, `task`, or `just`, and `file` is the file the
targets were read from. `resources` are the shared resources this project's
jobs need: two projects that name the same resource never run at the same
time. A job sent to a runner waits in that runner's own lane,
`<resource>@<runner>`, instead.

A target with no `estimate_seconds` has no usable history yet.
`estimate_level` says where the number came from — `exact` for that exact
command line, `target` or `project` when a shorter prefix of the history key
supplied it.

### status

The queue right now. Use it to see what else is going on, not to find out
whether your own job is done.

Input, all optional; `project` narrows the report to one project:

```json
{ "project": "apps" }
```

Output, with one job running and one item waiting behind it:

```json
{
  "actor": "claude-code",
  "running": [
    {
      "job_id": "J20260921_200346_9df790",
      "pid": 65170,
      "command": "--config /home/you/.config/skram/config.yaml run --foreground apps deploy --foreground --_job-id J20260921_200346_9df790",
      "project": "apps",
      "target": "deploy",
      "actor": "claude-code",
      "status": "running",
      "started_at": "2026-09-21T20:03:46.77215-04:00",
      "job_dir": "/home/you/.local/share/skram/logs/J20260921_200346_9df790",
      "elapsed_seconds": 4,
      "estimate_seconds": 12,
      "remaining_seconds": 8,
      "estimate_level": "target",
      "estimate_samples": 1,
      "estimate": "~8s remaining (est ~12s · from 1 run · target)",
      "item_id": "q_6a2071c8",
      "phase": "running",
      "resources": ["docker"]
    }
  ],
  "most_recent": {
    "id": "q_a71bf5ae",
    "command": "run --foreground apps deploy",
    "project": "apps",
    "target": "deploy",
    "actor": "claude-code",
    "status": "completed",
    "added_at": "2026-09-21T20:03:34.70192-04:00",
    "started_at": "2026-09-21T20:03:34.711815-04:00",
    "completed_at": "2026-09-21T20:03:46.73145-04:00",
    "job_id": "J20260921_200334_288ec1",
    "exit_code": 0,
    "duration_ms": 12019,
    "resources": ["docker"]
  },
  "queue": {
    "pending": 1,
    "running": 1,
    "completed": 4,
    "failed": 0,
    "killed": 0,
    "cancelled": 0,
    "processor_pid": 65618,
    "processor_alive": true,
    "by_actor": { "claude-code": { "running": 1, "pending": 1 } },
    "eta": "queue clears in ~9s (1 pending)",
    "eta_seconds": 9,
    "lanes": [
      {
        "resource": "docker",
        "running": [
          {
            "item_id": "q_6a2071c8",
            "job_id": "J20260921_200346_9df790",
            "project": "apps",
            "target": "deploy",
            "remaining_seconds": 8,
            "remaining": "8s"
          }
        ],
        "pending": 1,
        "eta": "~9s",
        "eta_seconds": 9
      }
    ]
  },
  "runners": {
    "studio": {
      "at": "2026-09-21T20:01:12.40312-04:00",
      "outcome": "ok",
      "version": "v0.21.0 (engine 0.21.0)",
      "os": "darwin",
      "arch": "arm64",
      "probed": true,
      "login_path": "/opt/homebrew/bin:/usr/bin:/bin",
      "missing_tools": ["pnpm"],
      "host_free_bytes": 128849018880,
      "docker_answered": true,
      "docker_engine": "OrbStack",
      "docker_context": "orbstack",
      "docker_free_bytes": 33285996544,
      "last_contact_seconds": 154
    },
    "old-box": { "outcome": "never" }
  }
}
```

The top-level `actor` is who is asking. `by_actor` is who is waiting, which is
the field that answers "is a person blocked behind me". `most_recent` is the
last item to finish, and `most_recent_artifacts` its job's artifacts archive
when it has one. `lanes` lists one entry per lane with anything live on it
(its `resource` is the lane's name, `cluster@studio` for jobs sent to a
runner), and the queue's `eta_seconds` is the longest lane's. Each running
job's `phase` is what it is doing now: `running` for a local job, and for one
sent to a runner `checking` (the runner's hello), `shipping` (pushing a
commit), `preparing` (resetting the runner's worktree), `running`, then
`homing` (bringing its files home), so a large push is not mistaken for a
hang. Either kind is in `setup` while the project's `setup:` target runs;
`queue list --json` carries the same `phase` on the running item, and the
job's end clears it. A running job sent to a runner carries `runner`; `leases`
lists the leases held on this machine, and `waiting_for_runner` and
`waiting_for_lease` the items waiting on a runner.

`runners` has one entry per declared [runner](runners.md), `{}` when there is
none. It is what this machine last heard from each, read from a file the queue
writes after every hello it makes (before a dispatch, while an item waits, and
in `runner check` and `doctor`), so `status` never contacts a runner. `at` is
that hello's time and `last_contact_seconds` its age. `outcome` is `ok` (it
answered), `unreachable` (it gave no answer, or none skram could read; `error`
says what happened), `declined` (its owner set `accept_runner_jobs: false`),
or `never` (no hello yet: the entry has nothing else). An answer also carries
the runner's `os`, `arch`, and Skram `version` (absent when it has none yet);
`probed` is true when it was asked about its tools, and then `login_path`,
`missing_tools`, and `missing_git` are what it said, as in [Checking a
runner](runners.md#checking-a-runner). `host_free_bytes` is the free space it
measured on the volume holding its Skram data, absent when it did not measure
it (an older Skram there), never zero in its place. A hello before a job of a
project that uses Docker (and `runner check` or `doctor` for a runner such a
project defaults to) also measures the runner's Docker engine (see [the Docker
engine](config.md#the-docker-engine)): `docker_answered` is whether it
answered, `docker_engine` its name (`Docker Desktop`, `OrbStack` or `Docker
Engine`), `docker_context` the `docker` context it was reached through,
`docker_error` its error line when it did not answer, and `docker_free_bytes`
the free space inside its storage. They are from the last hello that asked
about the engine, and `docker_at` is when that was: a later hello that did not
ask keeps them. Each is absent when no hello has measured it, and for an
unreachable runner. `skram runner list --json` carries the same fields for
each runner, and its plain output shows them as the `LAST PROBE`, `FREE DISK`
and `DOCKER` columns.

While the queue is on hold, `queue` carries a `hold` as well:

```json
{
  "pending": 1,
  "running": 0,
  "processor_alive": false,
  "hold": {
    "actor": "you",
    "reason": "cluster upgrade in progress",
    "since": "2026-09-21T20:04:00.803385-04:00"
  },
  "by_actor": { "claude-code": { "running": 0, "pending": 1 } },
  "eta": "queue clears in ~1s (1 pending)",
  "eta_seconds": 1
}
```

### run

The only way to run a target. Never call the project's script, `make`,
`kubectl`, or `docker build` directly; the guard stops those.

Input:

```json
{
  "project": "apps",
  "target": "test",
  "args": ["--scope", "auth"],
  "on_error": "stop",
  "checkout": "/home/you/dev/apps--T-42",
  "env": {"E2E_GREP": "login"}
}
```

`project` and `target` are required and come from `discover`. `args` are
handed to the target verbatim. `on_error` is `stop` — if this job fails,
cancel the items still pending in its lanes — or `continue`, which is two-way:
it also means this item, while still pending, survives a `stop` from a failure
ahead of it in its lane. Left out, the queue's own setting applies. `checkout`
runs the target in another checkout of the project's repo — a worktree or
another clone — instead of its configured path; absolute, or relative to the
directory the server was started in (never the caller's, since MCP has no
notion of "your cwd"). Left out, `run` uses the configured path, same as no
`-C` on the CLI. A `checkout` that is not a checkout of the project's repo is
refused, naming the project's repo and configured path. `env` is an object of
variables for this job alone, the same as `--env NAME=VALUE` on the CLI: set
last, over the project's `env:` and the server's environment, and refused when
a name is not an environment name or is one of the project's
[inputs](config.md#inputs) (see [per-job
variables](how-it-works.md#per-job-variables---env)).

`runner` sends the job to another machine, as `--runner` does on the CLI: the
name of an entry in the config's `runners:` (`local` runs it here). Left
out, the project's own `runner:` applies, if it has one. A name that is not
configured is an error result naming the ones that are, and an ephemeral
target with a named runner is refused. `wait`, `logs`, and `explain` then work
as for any job. See [Runners](runners.md).

On a target in the project's [`ledger:`](config.md#ledger) whose trees,
target, and args already have a green job, nothing is queued and the result is
the skip report the CLI's `run --json` prints: `skipped: true`, the green
`job_id`, `green_at`, `age_seconds`, and the `wait`, `explain`, and `force`
commands. `force: true` runs it anyway, as `--force` does (see [skipping a
proof that already
passed](how-it-works.md#skipping-a-proof-that-already-passed-ledger)).
`need_greens: N` skips only once the key has N consecutive greens since its
last red, as `--need-greens` does; each call queues at most one job. Pending
and running jobs of the key count toward N: once the greens plus those reach
it, nothing is queued and the result is `in_flight: true` with
`in_flight_items` and a `wait` command for the oldest (see [consecutive greens
and flakes](how-it-works.md#consecutive-greens-and-flakes---need-greens)).

For a project with [inputs](config.md#inputs) — other repos its targets
read — pass `checkouts` instead: a list of directories, one per repo,
resolved the same way (absolute, or relative to the server's own cwd) and
the same way the CLI's repeatable `-C` binds them, each classified by the
repo it is a checkout of. `checkout` and `checkouts` may not both be given;
that call is refused, saying to use one. Neither given, an input takes its
repo's default checkout; the server's own cwd is never followed for
`checkout`, `checkouts`, or a project's inputs, since it is the parent
session's, not the caller's:

```json
{
  "project": "apps",
  "target": "test",
  "checkouts": ["/home/you/dev/my-lib--x", "/home/you/dev/my-e2e--x"]
}
```

The call returns as soon as the item is queued. It does not wait:

```json
{
  "item_id": "q_6a2071c8",
  "project": "apps",
  "target": "deploy",
  "command": "run --foreground apps deploy",
  "actor": "claude-code",
  "pending": 1,
  "resources": ["docker"],
  "lanes": ["docker"],
  "lane_pending": 0,
  "estimate_seconds": 12,
  "estimate": "~12s (from 1 run · exact)",
  "queue_eta_seconds": 12,
  "wait": "skram wait q_6a2071c8"
}
```

`lane_pending` is how many items are already waiting in this one's lanes;
`queue_eta_seconds` adds the remainder of whatever is running there. `wait` is
the shell command that does what the `wait` tool does. Call `wait` with
`item_id` next. A `checkout` field, the checkout's absolute path, appears here
and in every later report naming this item or job (`wait`, `status`,
`explain`, `logs`) when the call named one; it is absent for a job in the
configured path. A project with inputs carries an `inputs` field the same
shape as the CLI's — keyed by variable, each with `repo`, `dir` (the checkout
the job read), and `default` — in this same set of reports; see
[inputs](config.md#inputs) for the full shape. It is absent for a project with
none. An `env` field, the variables the call gave, appears in the same reports
the same way and is absent when the call gave none.

While the queue is held, the same call still succeeds — the item is queued —
and says so:

```json
{
  "item_id": "q_41ea3ec9",
  "project": "apps",
  "target": "build",
  "actor": "claude-code",
  "pending": 1,
  "resources": ["docker"],
  "lanes": ["docker"],
  "lane_pending": 0,
  "estimate_seconds": 1,
  "queue_eta_seconds": 1,
  "held": true,
  "hold": {
    "actor": "you",
    "reason": "cluster upgrade in progress",
    "since": "2026-09-21T20:04:00.803385-04:00"
  },
  "wait": "skram wait q_41ea3ec9"
}
```

`held: true` means a person put the queue on hold, and `hold.reason` is what
they said. Wait, or stop and report the reason. Retrying, enqueueing
more, and running the target by hand are all wrong answers.

An ephemeral target is read-only: it has no lane, no item, and no job
directory, so `run` executes it immediately and returns what it printed.

```json
{
  "ephemeral": true,
  "project": "apps",
  "target": "status",
  "exit_code": 0,
  "output": "nothing running\n"
}
```

It carries `checkout` and `env` too when the call gave them, and `inputs` for a
project that has them; the child sees each input's variable and its
`SKRAM_INPUT_<VAR>_DEFAULT` companion exactly as a queued run's would.

### wait

Block until something finishes. This is the reliable way to learn an outcome.

Input, all optional:

```json
{ "ref": "q_6a2071c8", "timeout_seconds": 120 }
```

`ref` is an item id (`q_…`) or a job id (`J…`). With no `ref` it waits for the
whole queue; with `resource` it waits for one lane instead (a resource name,
or `<resource>@<runner>`), and `ref` and `resource` cannot both be given.
`timeout_seconds` defaults to 1500 (25 minutes) and is capped at 1500, so the
call always returns before an MCP client gives up on a tool that has sent
nothing (Claude Code aborts one after 30 minutes). A larger value is capped, and
the result names the limit it used as `timeout_seconds`, with
`timeout_capped: true`. A job longer than that takes more than one call: see
the `timeout` row below. The `skram wait` command is unchanged and takes any
`--timeout`.

When the request carries a progress token, `wait` sends a progress
notification every 30 seconds while it blocks. Its `message` names the item's
phase (`pending`, `waiting_for_runner`, `waiting_for_lease`, or what a running
job is doing: `checking`, `shipping`, `preparing`, `running`, `homing`) and the
seconds elapsed in the call, for example `shipping, 90s elapsed`. A request
without a token gets none.

Output for an item that finished:

```json
{
  "kind": "item",
  "item_id": "q_6a2071c8",
  "job_id": "J20260921_200346_9df790",
  "project": "apps",
  "target": "deploy",
  "status": "completed",
  "exit_code": 0,
  "duration_ms": 12021,
  "job_dir": "/home/you/.local/share/skram/logs/J20260921_200346_9df790"
}
```

A job with an artifacts archive (the project's
[`artifacts:`](config.md#artifacts-and-artifacts_limit)) adds `"artifacts"`,
its path in the job directory; the key is absent when there is none.

`status` is one of:

| `status` | What happened |
| --- | --- |
| `completed` | the target ran and exited 0 |
| `failed` | it ran and exited non-zero; `exit_code` is the target's own |
| `cancelled` | the item never ran; `cancelled_by` says who or what withdrew it |
| `killed` | the job was stopped while it was running, by `skram kill` or any other signal; `exit_code` is 137 |
| `timeout` | `timeout_seconds` elapsed and it is still going: call `wait` again with the same `ref` |

A timeout is not a failure and not an error — the job is still going, and
calling `wait` again with the same `ref` is the right response, never `run`
again. The result carries `still_running: true`, `phase` (as above),
`elapsed_seconds` (spent in this call), and a `message` saying to call `wait`
again. An item still waiting for its runner to answer, or for a lease on it,
carries `waiting_for_runner` or `waiting_for_lease`; that is expected, so wait
again rather than queueing it a second time:

```json
{
  "kind": "item",
  "item_id": "q_6a2071c8",
  "job_id": "J20260921_200346_9df790",
  "project": "apps",
  "target": "deploy",
  "status": "timeout",
  "timeout_seconds": 1500,
  "still_running": true,
  "phase": "shipping",
  "elapsed_seconds": 1500,
  "message": "still running: call wait again with ref q_6a2071c8; do not run it again (shipping, 1500s elapsed)"
}
```

A cancelled item never produced a job, so there is no exit code and no job
directory:

```json
{
  "kind": "item",
  "item_id": "q_41ea3ec9",
  "project": "apps",
  "target": "build",
  "status": "cancelled",
  "cancelled_by": "claude-code"
}
```

Waiting for the whole queue or for one lane answers with `kind` `queue` or
`lane` instead of `item`:

```json
{ "kind": "queue", "status": "idle", "waited_items": 1 }
```

```json
{ "kind": "lane", "resource": "docker", "status": "idle" }
```

### explain

Read this after a failure, before `logs`. It is one report built from the job's
recorded status, the queue, the job's events, and the estimate history, and it
ends in `next`: the commands to run.

Input, all optional — no `ref` means the most recent job, and `lines`
(default 40) is how much of the stderr tail to include:

```json
{ "ref": "q_3e419ece", "lines": 5 }
```

Output for a job that failed:

```json
{
  "job_id": "J20260921_200400_c8e8d2",
  "item_id": "q_3e419ece",
  "project": "apps",
  "target": "test",
  "actor": "claude-code",
  "resources": ["docker"],
  "lanes": ["docker"],
  "status": "failed",
  "exit_code": 1,
  "duration_ms": 16,
  "reason": "test_failure",
  "reason_detail": "FAIL: 1 test failed",
  "tail_stream": "all",
  "tail": [
    "bash /home/you/dev/apps/ops.sh test",
    "running tests",
    "FAIL: 1 test failed"
  ],
  "next": [
    "fix the failing test, then: skram run apps test",
    "full log: skram logs J20260921_200400_c8e8d2"
  ],
  "job_dir": "/home/you/.local/share/skram/logs/J20260921_200400_c8e8d2"
}
```

`reason` is the classifier's verdict on what the target printed, or what Skram
found before or around it. The classifier reads the last 200 lines of each
stream against one ordered list of rules, and the first rule with a matching
line wins:

| Reason | Matched on |
|---|---|
| `disk_full` | `no space left on device` (also set before the job starts, when the machine is short of disk: its host disk, or the Docker engine's storage for a project that uses Docker) |
| `docker_down` | `Cannot connect to the Docker daemon`, `failed to connect to the docker API`, `Docker Desktop is unable to start` (also set before the job starts, when the Docker engine of a project that uses Docker does not answer) |
| `test_failure` | a test runner's summary: Playwright's `N failed`, Jest's or Vitest's `Tests: N failed`, pytest's `=== N failed … ===`, Go's `--- FAIL:` or `FAIL <package> <time>`; else `AssertionError` or `N tests failed` |
| `script_error` | a missing command or module (`command not found`, pnpm's `Command "…" not found`, `Cannot find module`), then a shell syntax error, `No such file or directory`, `bad interpreter`, make's `No rule to make target` |
| `dependency` | Go module and npm/pnpm fetch errors, `could not resolve host`, `TLS handshake timeout`, `connection reset by peer` |
| `image_pull` | `ErrImagePull`, `ImagePullBackOff`, `pull access denied`, `manifest unknown` |
| `k8s_rollout` | `exceeded its progress deadline`, `CrashLoopBackOff`, `error: timed out waiting for the condition` |
| `lint` | a linter's summary or finding: golangci-lint, ESLint, flake8/ruff, RuboCop, Prettier, `vet:` |
| `compilation` | a compiler's `file:line:col:` error (Go, C, Java, Swift), `undefined: …`, TypeScript's `error TS…`, rustc's `error[E…]`, `SyntaxError` |
| `docker_build` | `failed to solve:` |
| `permission` | `permission denied`, `access denied`, `EACCES`, `operation not permitted` |
| `unknown` | nothing matched |

Rules match whole words or a line's start, so a log that only mentions
`expected`, `lint` or `tsc` in passing is not classified by them. Reasons
Skram sets itself: for a job sent to a runner `runner_unreachable`,
`runner_refused`, `runner_lost`, or `runner_declined`; `setup_failed` when
the project's `setup:` target failed before the target started;
`missing_tool` when a command the project's `tools:` names was not on that
machine's `PATH` and the target never started; `disk_full` when the machine
it would start on was short of disk before it began; `checkout_removed`
when the checkout was gone before the job started; `start_failed` for a job
that ended before its target ran; and `signal` for one killed by a signal
from outside `skram kill`. `disk_full`, `docker_down`, `missing_tool`,
`setup_failed` and `start_failed` say the machine was unfit, not that the
branch is wrong: a job that fails with one is recorded neither green nor red
on a [ledgered](how-it-works.md#skipping-a-proof-that-already-passed-ledger)
target.

`reason_detail` is the line the rule matched; for `test_failure`, the test
runner's summary line when it printed one. For `unknown` it is the last line
that looks like an error (`error`, `fatal`, `failed`, `panic`), else the last
stderr line; for `start_failed`, the error that stopped it. `tail_stream` is
`stderr` when the job wrote any and `all` when it did not. `artifacts` is the
job's artifacts archive, absent when it has none.

A job still running explains itself too, with the estimate and how to follow
or stop it:

```json
{
  "job_id": "J20260921_200346_9df790",
  "item_id": "q_6a2071c8",
  "project": "apps",
  "target": "deploy",
  "actor": "claude-code",
  "resources": ["docker"],
  "lanes": ["docker"],
  "status": "running",
  "estimate_seconds": 12,
  "tail_stream": "all",
  "tail": ["bash /home/you/dev/apps/ops.sh deploy", "deploying"],
  "next": [
    "follow: skram logs J20260921_200346_9df790 -F",
    "stop: skram kill J20260921_200346_9df790"
  ],
  "job_dir": "/home/you/.local/share/skram/logs/J20260921_200346_9df790"
}
```

An item that has not started gives its `position` in the queue and, when the
queue is on hold, who held it and what they said:

```json
{
  "job_id": "",
  "item_id": "q_41ea3ec9",
  "project": "apps",
  "target": "build",
  "actor": "claude-code",
  "resources": ["docker"],
  "lanes": ["docker"],
  "status": "pending",
  "tail": [],
  "next": [
    "held by you: cluster upgrade in progress; wait: skram wait q_41ea3ec9 --timeout 30m; or ask the holder to skram release"
  ],
  "hold": {
    "actor": "you",
    "reason": "cluster upgrade in progress",
    "since": "2026-09-21T20:04:00.803385-04:00"
  },
  "position": 1
}
```

### logs

The last N events of a job's `events.jsonl`, as raw objects. Reach for it when
`explain`'s tail was not enough.

Input, all optional — no `ref` means the most recent job, an item id resolves
to the job it produced, and `tail` defaults to 40:

```json
{ "ref": "q_f9fa2777", "tail": 6 }
```

Output:

```json
{
  "job_id": "J20260921_200358_dced18",
  "job_dir": "/home/you/.local/share/skram/logs/J20260921_200358_dced18",
  "events": [
    {
      "ts": "2026-09-21T20:03:58.794703-04:00",
      "job": "J20260921_200358_dced18",
      "project": "apps",
      "target": "build",
      "step": "build",
      "step_id": "build/apps",
      "phase": "job_start",
      "level": "info",
      "msg": "Running apps/build"
    },
    {
      "ts": "2026-09-21T20:03:58.799377-04:00",
      "job": "J20260921_200358_dced18",
      "project": "apps",
      "target": "build",
      "step": "build",
      "step_id": "build/apps",
      "phase": "log",
      "level": "info",
      "msg": "building",
      "stream": "stdout",
      "line": "building",
      "meta": {
        "cmd": "bash /home/you/dev/apps/ops.sh build",
        "cwd": "/home/you/dev/apps",
        "pid": 70492
      }
    },
    {
      "ts": "2026-09-21T20:03:59.806944-04:00",
      "job": "J20260921_200358_dced18",
      "project": "apps",
      "target": "build",
      "step": "build",
      "step_id": "build/apps",
      "phase": "job_end",
      "level": "info",
      "msg": "Finished apps/build: success (exit 0, 1.012s)",
      "status": "success",
      "duration_ms": 1012
    }
  ]
}
```

The phases run `job_start` → `step_start` → `log` → `step_end` → `job_end`.
An output line carries `stream` (`stdout`, `stderr`, or `skram` for the
command line the job ran) and `line`. Never read a job's `raw.log` whole.

## Errors

A tool that cannot do what was asked returns an error result whose text says
what was wrong and what the valid values were, rather than an empty answer:

```json
{
  "content": [{ "type": "text", "text": "unknown project \"nope\" (projects: apps)" }],
  "isError": true
}
```

## Resources

Two kinds, both `text/markdown`:

| URI | What it is |
| --- | --- |
| `skram://guide` | the agent rules: the loop, backing off when the queue is held, ephemeral targets, the guard, where a job's files live. Byte for byte the text `install-rules` writes into a checkout. |
| `skram://projects/<name>` | one project's full target table |

Every configured project is listed as its own resource, and
`skram://projects/{name}` is offered as a template as well, so a client that
resolves templates can read a project the listing did not name.

A project resource reads like this:

```text
### Targets: apps

| Target | What it does | Typical | Notes |
|---|---|---|---|
| `build` | build the image | ~1s |  |
| `deploy` | roll the new image out | ~12s |  |
| `status` | print what is running |  | ephemeral: runs immediately, no job |
| `test` | run the test suite |  | no history yet |

Entry point: `ops.sh` (script) in `/home/you/dev/apps`. Run: `skram run apps <target> [args] --json`.
Resource lane: `docker`. Jobs here serialise with every other project on the same lane.
```

A project with `inputs:` gets one more line, naming each variable and repo
— never a machine path, so the table stays the same on every machine — for
example `` Inputs: `LIB_DIR` (repo `my-lib`), `E2E_DIR` (repo `my-e2e`). ``

## The job-end notification

After `run` enqueues an item the server watches it and, when it ends, sends the
client one logging notification carrying the object `wait` would have returned.

This is best effort and nothing should depend on it. The notification is
dropped unless the client has set a logging level, and under recent protocol
versions that level is scoped to a single request, so a notification sent after
the `run` call returned is dropped by design. The watcher also gives up after
four hours.

`wait` is the reliable path. Treat a notification as an early hint, never as
the answer.
