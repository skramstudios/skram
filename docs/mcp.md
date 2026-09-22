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
Call it first whenever a project or target name is not certain.

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
  }
}
```

`backend` is `script`, `make`, `task`, or `just`, and `file` is the file the
targets were read from. `resources` are the lanes this project's jobs occupy:
two projects that name the same resource never run at the same time.

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
  }
}
```

The top-level `actor` is who is asking. `by_actor` is who is waiting, which is
the field that answers "is a person blocked behind me". `most_recent` is the
last item to finish. `lanes` lists one entry per resource with anything live on
it, and the queue's `eta_seconds` is the longest lane's.

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
  "on_error": "stop"
}
```

`project` and `target` are required and come from `discover`. `args` are
handed to the target verbatim. `on_error` is `stop` — if this job fails,
cancel the items still pending in its lanes — or `continue`; left out, the
queue's own setting applies.

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
`item_id` next.

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

### wait

Block until something finishes. This is the reliable way to learn an outcome.

Input, all optional:

```json
{ "ref": "q_6a2071c8", "timeout_seconds": 120 }
```

`ref` is an item id (`q_…`) or a job id (`J…`). With no `ref` it waits for the
whole queue; with `resource` it waits for one lane instead, and `ref` and
`resource` cannot both be given. `timeout_seconds` defaults to 1800 and is
capped at 3600; a good choice is `run`'s `estimate_seconds` doubled, at least
600.

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

`status` is one of:

| `status` | What happened |
| --- | --- |
| `completed` | the target ran and exited 0 |
| `failed` | it ran and exited non-zero; `exit_code` is the target's own |
| `cancelled` | the item never ran; `cancelled_by` says who or what withdrew it |
| `killed` | the job was stopped while it was running |
| `timeout` | `timeout_seconds` elapsed and it is still going |

A timeout is not a failure and not an error — the job is still going, and
calling `wait` again is the right response:

```json
{
  "kind": "item",
  "item_id": "q_6a2071c8",
  "job_id": "J20260921_200346_9df790",
  "project": "apps",
  "target": "deploy",
  "status": "timeout"
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

`reason` is the classifier's verdict — `test_failure`, `compilation`,
`dependency`, `docker_build`, `k8s_rollout` and so on — and `reason_detail` is
the line it keyed on. `tail_stream` is `stderr` when the job wrote any and
`all` when it did not.

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
