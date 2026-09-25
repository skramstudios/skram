# Configuration reference

Skram reads one YAML file. This page lists every key it accepts, the default
it uses when the key is absent, and what changes when you set it.

The file is shared with the rest of the family: Skram reads `projects:`,
`workflows:`, `reaper:`, `repos:`, `logs_dir:`, and `queue_dir:`, and ignores
every other top-level block except the removed ones at the end of this page.
Only `projects:` is required.

- [Where the file is](#where-the-file-is)
- [`projects:`](#projects)
- [`workflows:`](#workflows)
- [`reaper:`](#reaper)
- [`repos:`](#repos)
- [`logs_dir:` and `queue_dir:`](#logs_dir-and-queue_dir)
- [Blocks another product owns](#blocks-another-product-owns)
- [Keys that are now errors](#keys-that-are-now-errors)
- [Checking the file](#checking-the-file)

## Where the file is

`~/.config/skram/config.yaml`, or `$XDG_CONFIG_HOME/skram/config.yaml` when
that variable is set.

```bash
skram init                      # write a starter file at the default path
skram init --detect             # add the checkout you are in as a project
skram --config ./other.yaml status
```

`--config` works on every command. The queue processor is started with the
absolute path of the file the item was enqueued from, and that path is passed
to the job, so a job always runs against the configuration that queued it.

The file is read fresh on every command. A bad value stops the command before
anything is enqueued, and the message names the project and the key.

`~/` at the start of a value expands to your home directory. This applies to a
project's `path:`, the paths under `repos:`, `logs_dir:`, and `queue_dir:`.
Nothing else is expanded: environment variables and `$HOME` are literal text.

## `projects:`

A project is a configured place to run targets. The key is the project's name,
which is what you type in `skram run <project> <target>` and what appears in
the queue.

```yaml
projects:
  my-app:
    path: ~/dev/my-app
    script: ops.sh
  api:
    path: ~/dev/api
    makefile: Makefile
  web:
    path: ~/dev/web
    taskfile: Taskfile.yml
  infra:
    path: ~/dev/infra
    justfile: justfile
```

### `path:`

Required. The directory targets run in. It must be absolute once `~/` is
expanded; a relative path is refused with `project "my-app" path must be
absolute`. The directory does not have to exist when the file is read — a
missing one is reported by `skram doctor`, not by the loader.

### `script:`, `makefile:`, `taskfile:`, `justfile:`

Exactly one of the four. The value is the file targets come from, relative to
`path:` unless it is absolute. Setting none is an error naming all four;
setting two is `project "my-app" sets both makefile and justfile; pick one`.

Targets are read by parsing that file. Skram never runs the file to list them,
so discovery works in a shell hook and on a machine that does not have `make`,
`task`, or `just` installed. Includes and imports are not followed.

| Key | Targets come from | A target runs as |
| --- | --- | --- |
| `script:` | the arms of a column-0 `case … in` block, one-line (`deploy) ./do-deploy.sh ;;`) or multi-line, quoted or bare, `a\|b)` alternatives each their own target; a trailing `# comment` on the arm (after `;;` for a one-line arm) is the help text | `bash <file> <target> [args]` |
| `makefile:` | every name in a `.PHONY:` line and every `target: ## help` rule; with neither, every plain-name rule | `make -f <file> <target> [args]` |
| `taskfile:` | the top-level `tasks:` keys, minus `internal: true`; `desc:` is the help | `task --taskfile <file> <target> [-- args]` |
| `justfile:` | column-0 recipes, minus `[private]` and `_`-prefixed names; the comment run above is the help | `just --justfile <file> <target> [args]` |

Arguments you add after the target are passed through unchanged.

### `resource:`

The shared thing this project's targets must not touch at the same time as
another job: a cluster, a Docker VM, a deploy target. Each resource has one
lane, and a lane runs one job at a time.

```yaml
projects:
  api:
    path: ~/dev/api
    makefile: Makefile
    resource: kind-dev
  web:
    path: ~/dev/web
    taskfile: Taskfile.yml
    resource: [kind-dev, docker]
```

Default: the project's own name, so an unconfigured project takes turns only
with itself. A scalar or a list; a list means the item waits for every one of
those lanes and holds them all while it runs. Two projects that name the same
resource share one lane, which is how you stop two checkouts from building
against the same cluster.

A name must match `^[a-z0-9][a-z0-9._-]*$`, so it can be used as a flag value
and a JSON key. Anything else is refused at load. Names are sorted and
de-duplicated.

### `ephemeral:`

Targets that are read-only and may run immediately: no lane, no item, no job
directory, no history.

```yaml
projects:
  my-app:
    path: ~/dev/my-app
    script: ops.sh
    expose: [build, deploy, status]
    ephemeral: [status]
```

Default: empty — every target is queued. An ephemeral target runs immediately
whether or not `--now` is passed, and `--now` on any other target is an error
that tells you to attach with `-f` instead. The guard also lets an ephemeral
target through when it sees it on a command line.

Keep the list to targets that only read. A target listed here bypasses the
lane, so a write in it can land beside a running job.

### `expose:`

The targets this project offers.

Default: empty, which means every target found in the file, in alphabetical
order. When the list is not empty it is a filter and an order: `skram run`,
`skram discover`, the MCP server, the TUI, and the agent tables show only
these, in the order listed, and a name outside the list is not a target you
can run. A name that is not among the file's targets is reported
by `skram doctor`, as is an ephemeral target that a non-empty `expose:` leaves
out (it could never run).

### `env:`

Extra environment variables for every job of this project.

```yaml
projects:
  my-app:
    path: ~/dev/my-app
    script: ops.sh
    env:
      NODE_ENV: production
      DOCKER_BUILDKIT: '1'
```

Default: empty. The values are added on top of the environment the job
inherits, so they win over a variable of the same name. They reach queued
jobs, attached jobs, ephemeral targets, workflow steps, and runs started
through the MCP server. Values are literal: nothing inside them is expanded.
One run can override a value for itself alone with
[`skram run --env NAME=VALUE`](how-it-works.md#per-job-variables---env),
which is set after `env:` and wins.

### `inputs:`

The other repos this project's targets read, one environment variable each.
A project whose script builds from one repo and tests from another names both
here instead of pinning their directories as literals in `env:`.

```yaml
projects:
  my-app:
    path: ~/dev/my-app
    script: ops.sh
    inputs:
      LIB_DIR: my-lib
      E2E_DIR: my-e2e
repos:
  my-lib:
    remote: my-org/my-lib
    paths: [~/dev/my-lib, ~/work/my-lib]
  my-e2e:
    paths: [~/dev/my-e2e]
```

Default: empty. The key is the variable, the value the name of an entry under
[`repos:`](#repos). An input is always the repo's root; a subdirectory is the
script's business. Each input has a default directory: the first of the repo's
`paths:` (after `~/` and glob expansion) that exists as a directory on this
machine, so one file works on two machines that keep their checkouts in
different places.

The loader refuses, naming the project and the variable:

- a repo that is not under `repos:`, or one with no `paths:`;
- a variable that `env:` also sets, since the file would have two answers
  for it;
- a variable that is not an environment name (`^[A-Za-z_][A-Za-z0-9_]*$`).

A repo none of whose paths exists here is not a load error, since another
machine may have it: `skram doctor` reports it, one line per project.
`skram discover` lists each input under its project with the default
directory; `skram discover --json` carries them as `inputs`, keyed by
variable, each with `repo`, `default`, and `error` when there is no default.

A job follows each input the way it follows the project's own checkout: to
the checkout of that repo you stand in, to one you name with `-C <dir>`
(repeatable, one per repo, and it beats the directory you stand in), and
otherwise to the default. So from a worktree of `my-lib`,
`skram run my-app build` builds that worktree, and
`skram run my-app test -C ../my-lib--x -C ../my-e2e--x` binds both inputs
from anywhere. See [running in another checkout](how-it-works.md#running-in-another-checkout--c).

`skram discover` follows inputs the same way: run from a checkout of one, or
with `-C`, it prints the same `input VAR → dir (default …)` line `run`
prints for each input that resolves off its default, and `skram discover
--json` adds `dir` (absolute) to that input's object. Nothing extra when
every input is on its default. The `### Targets: <project>` table
`install-rules` writes, and the matching `skram://projects/<name>` MCP
resource, carry one more line for a project with inputs — `` Inputs:
`LIB_DIR` (repo `my-lib`), `E2E_DIR` (repo `my-e2e`). `` — naming each
variable and repo, never a machine path, so the table is still a function
of your config alone.

The target sees each variable set to the directory it resolved to, and
`SKRAM_INPUT_<VAR>_DEFAULT` (for example `SKRAM_INPUT_LIB_DIR_DEFAULT`) set to
the default directory, so a script can fall back to the default checkout for
files a worktree does not have. Both are set after `env:` and the environment
the job inherits, so they win. This holds for queued, attached (`-f`), and
ephemeral runs. `skram run --env` refuses an input's variable: `-C` is the
only way to move an input.

A run is refused, before anything is queued, when an input has no default
and nothing named a checkout of its repo. `skram run` prints one line on
stderr for each input that is off its default,
`input LIB_DIR → /home/me/dev/my-lib--x (default /home/me/dev/my-lib)`, and
nothing when every input is on its default. Inputs never create a lane or
change a target's estimate history.

Every job records where its inputs resolved. The queue item, the job's
`status.json`, its `job_start` and `job_end` events, and every `--json`
report that names the job or item (`run`, `status`, `queue list`, `wait`,
`explain`, `logs`) carry `inputs`, keyed by variable, each with `repo`,
`dir` (the absolute checkout the job read), `default` (the repo's
default directory, absent when it has none on this machine), `head` (the
commit `dir` was on when the job was queued), and `head_at_start` (the
commit it was on when the job started; absent until then). An input on its
default has `dir` equal to `default`. The key is absent for a project with
no inputs:

```json
"inputs": {
  "E2E_DIR": { "repo": "my-e2e", "dir": "/home/me/dev/my-e2e", "default": "/home/me/dev/my-e2e",
               "head": "3f9c…", "head_at_start": "3f9c…" },
  "LIB_DIR": { "repo": "my-lib", "dir": "/home/me/dev/my-lib--x", "default": "/home/me/dev/my-lib",
               "head": "a1b2…", "head_at_start": "c3d4…" }
}
```

Where each input resolved is fixed when the job is queued: editing `paths:`
while it waits does not move it. What is in that directory is read when the
job starts, so a commit or checkout in an input's directory while the job
waits changes what it builds. When `head_at_start` differs from `head` the job still runs, and
its log gets one warning line per moved input, also written as a `warn`
event tagged `input_drift`:

```text
[skram] WARNING: input LIB_DIR HEAD moved between enqueue and start: a1b2… → c3d4… (/home/me/dev/my-lib--x); the job runs what is there now
```

`explain` shows the move on that input's line under `Inputs:`, lists the
moved variables as `inputs_drifted` in `--json`, and after a failure adds a
hint naming the commit the job actually ran. This covers an input on its
default too: the configured checkout switching branch under a pending job
is visible the same way. An input whose directory is gone (or is no longer
a git checkout) when the job's turn comes fails the job without running,
with the reason `checkout_removed` and a detail naming the variable and the
directory; `explain` suggests re-running with `-C` naming another checkout
of that repo, or its default.

Human output names each input that is off its default by its directory
name, after the project's own checkout when that is off `path:` too, in
variable order: `my-app/build (my-lib--x)`, or
`my-app/build (my-app--feature, my-lib--x)`. When every input is on its
default the label is the same as for a project without inputs. `status`,
`queue list`, `logs -F`, `explain`, `skram tui`, the dashboard, and the
[herdr](herdr.md) sidebar all use this label; `explain` also lists every
input with its directory and default under `Inputs:`.

### `guard:`

The guard is a shell hook. It reads a command an agent is about to run and
blocks the ones that would bypass the queue: a project's ops script anywhere
(unless the target is ephemeral), a `make`, `task`, or `just` target of a
project that applies where the command runs and is backed by that file, and
`kubectl` mutations, `docker build`, and `compose up`/`down` anywhere. It
fails open: anything it cannot judge is allowed.

```yaml
projects:
  my-app:
    path: ~/dev/my-app
    script: ops.sh
    guard:
      enabled: true
      allow:
        - '^kubectl get '
        - '^docker compose ps'
```

`allow:` is a list of RE2 regular expressions. Default: empty. Each pattern is
matched against the single command and against the whole command line, so a
pattern can exempt one command in a chain. The allow list of a project applies
where that project applies: inside a checkout the `repos:` block ties it to,
or one that contains the project's `path:`. It exempts the `kubectl`, `docker`,
and `make`/`task`/`just` rules; a configured ops script is blocked regardless.

`enabled:` defaults to true. `false` takes the project out of the guard: its
ops script, its `make`/`task`/`just` targets, and its `allow:` list are all
ignored. The `kubectl` and `docker` rules belong to no project and still
apply; use `allow:` on a project that applies there for a command you want
through. One `SKRAM_*` variable turns the guard off in a shell, and
[troubleshooting](troubleshooting.md) lists it; registering the hook is on the
[agent setup](agent-setup.md) page.

### Keys Skram loads and does not act on

Two project keys are parsed, validated as YAML, and then left alone by every
`skram` command. They are read by programs that embed Skram's queue and
register verbs of their own. Set them only if something you run asks for them.

```yaml
projects:
  my-app:
    path: ~/dev/my-app
    script: ops.sh
    commands:
      deploy:
        args: [deploy, --wait]
      full-setup:
        steps:
          - label: create the cluster
            args: [cluster-create]
            flag: with-cluster
            optional: false
      release:
        workflow: deploy-stack
    service_routes:
      web: my-app
```

- `commands:` — named argument lists for an embedder's own verb. An entry
  takes `args:`, the arguments handed to the project's file; or `steps:`, a
  list run in order, each with its own `args:`, an optional `label:` for the
  log, `optional:` to let a failure pass, and `flag:` naming a switch that
  must be on for the step to run; or `workflow:`, naming an entry of the
  `workflows:` block.
- `service_routes:` — a map an embedder can use to route a name to a project.
  Nothing in Skram reads it.

## `workflows:`

A workflow is several targets with a dependency graph between them.

```yaml
workflows:
  deploy-stack:
    steps:
      - project: infra
        target: cluster-create
      - project: api
        target: deploy
        depends_on: [0]
        args: [--wait]
      - project: web
        target: deploy
        depends_on: [0]
        optional: true
        flag: with-web
```

```bash
skram workflow list                          # the workflows this file defines
skram workflow run deploy-stack              # run one
```

Default: no workflows. Each step takes:

| Key | Default | What it does |
| --- | --- | --- |
| `project:` | required | the project the step runs in; it must exist |
| `target:` | required | the target to run |
| `args:` | empty | extra arguments after the target; `{name}` is replaced by the value of a matching `--var name=value` |
| `depends_on:` | empty | indices of steps that must finish first, counting from `0` in the order written |
| `optional:` | `false` | a failure is logged and the workflow continues |
| `flag:` | empty | the step runs only when `--flag <name>` is passed; otherwise it is skipped and its dependants still run |

Steps with no unmet dependency run in parallel. An index outside the list, a
step that depends on itself, a cycle, an empty step list, and a step naming an
unknown project are all refused before anything runs.

`skram workflow run` queues the workflow as one job. The item names every
lane its steps' projects declare (`resource:`, or the project's own name), so
it starts when all of those lanes are free and holds all of them until its
last step ends; no queued job runs beside a step on a resource the step's
project protects. Inside the job, steps run in parallel where the graph
allows. The item carries the actor, obeys a hold, and takes `-f`, `--json`,
and `--on-error` like `skram run`:

```bash
skram workflow run deploy-stack -f            # attach; the exit code is the workflow's
skram workflow run deploy-stack --json        # the enqueue report, with its lanes
skram wait --last                             # or wait on the item id
skram workflow run deploy-stack -C ../api--feature   # api's steps in that worktree
skram workflow run deploy-stack -C ../lib--x -C ../e2e--x   # each step's inputs follow
```

Run from inside another checkout of a step's repo (a worktree, a second
clone), the steps whose project is in that repo run there, from that
checkout's own backend file, and every other step runs in its `path:`;
`-C, --checkout <dir>` names the checkout explicitly. The item and
`status.json` then carry `checkout`, and the workflow shows as
`_workflow/<name> (<checkout dir name>)`; when steps of two repos each run in
a checkout of their own, `checkout` names the first step's.

Steps follow [inputs](#inputs) the same way. `-C` may be given more than once,
one directory per repo, and each step resolves its own project's checkout and
inputs from that one list and the directory you stand in, exactly as
`skram run` does for that project: a `-C` of a repo the step's project
neither lives in nor reads is ignored for that step, and a step whose project
has no inputs is untouched. Each step's child gets its own input variables
and `SKRAM_INPUT_<VAR>_DEFAULT`, as a `skram run` child does. A `-C` that no
step's repo or inputs can use is refused, naming the steps' projects and
their input repos. The item and `status.json` carry `inputs`, the union of
every step's; two steps whose projects bind one variable to different
checkouts (the same variable name on two different repos) are refused at
enqueue, since the record has one answer per variable: rename the input in
one of the projects.

The job's directory under `logs_dir:` has an event per step, its
`status.json` names the workflow as `_workflow/<name>`, and its exit code is
the first failing step's.

## `reaper:`

Repeated image builds fill the Docker disk. The reaper removes old timestamped
image tags and prunes build caches to a budget. It touches the Docker daemon
on this machine only: no registry, no cluster runtime, no volume.

```yaml
reaper:
  enabled: true
  keep_tags: 2
  tag_pattern: '^\d{8}-\d{6}$'
  cache_budget: 20GB
  image_repos:
    - 'localhost:6001/*'
  builders:
    - multiarch
  after_targets:
    - 'my-app/build*'
  timeout: 5m
```

| Key | Default | What it does |
| --- | --- | --- |
| `enabled:` | `false` | turns on the automatic reap after a job. It does not gate the command: `skram reap` runs either way |
| `keep_tags:` | `2` | how many matching tags to keep per image repository, newest first |
| `tag_pattern:` | `^\d{8}-\d{6}$` | the regular expression a tag must match to be eligible. The default matches timestamp tags such as `20260921-143005`, so `latest` and version tags never match and are never removed |
| `image_repos:` | empty | glob patterns for the image repositories to look at. Nothing is reaped until this is set |
| `cache_budget:` | `20GB` | the build cache each builder is pruned down to |
| `builders:` | empty | extra buildx builders to prune besides the default one, which is always pruned. A builder that is not present is skipped |
| `after_targets:` | empty | `project/target` glob patterns. After a job whose project and target match one, and only when it succeeded, a reap runs inside that job |
| `timeout:` | `5m` | the budget an automatic reap runs its whole pass under, and the per-call cap on every destructive docker call it makes (`rmi`, and the image/builder/buildx prunes — a big removal or a cache prune down to budget can legitimately take a while). A read-only call (`image ls`, `buildx inspect`, `system df`) is capped at a shorter, fixed 30s instead, so a wedged Docker daemon is killed rather than left to hang the job's lane. A `system df` past its cap only skips the disk usage report. On any other expiry the reap logs which step timed out and stops there, and the job's own exit code is unchanged |

```bash
skram reap --dry-run            # list what would be removed
```

An automatic reap writes its output into the job's log and never fails the
job. Removing a tag that is in use is logged and skipped. A structural problem
— an unparseable `tag_pattern:`, a Docker daemon that cannot be reached, or no
`image_repos:` — fails the command, and is a log line when the reap runs after
a job. One `SKRAM_*` variable skips automatic reaps without editing the file;
[troubleshooting](troubleshooting.md) lists it.

A docker call that hangs past its cap is killed either way — the 30s query
cap or, for a removal or a prune, `timeout:`. Run by hand, `skram reap`
reports that expiry as an error and exits non-zero; inside a job it is the
same "structural problem" case above, so it stops the reap and the automatic
pass logs one line naming the step, without failing the job.

## `repos:`

A repo is one upstream, with the checkouts of it on this machine. The block
says what is true inside those checkouts: which projects apply there, and
which vault namespaces are in scope.

```yaml
repos:
  my-app:
    remote: my-org/my-app
    paths: [~/dev/my-app, ~/work/my-app]
    projects: [api, web]
    default: api
  local-ops:
    paths: [~/dev/ops-*]
    projects: [infra]
```

The entry name must match `^[a-z0-9][a-z0-9-]*$`. An entry needs `remote:` or
at least one path.

| Key | Default | What it does |
| --- | --- | --- |
| `remote:` | none | the upstream: `owner/name` (GitHub assumed) or any git URL. It is matched against a checkout's `origin` after normalising, so the SSH and HTTPS spellings, with or without `.git`, are the same repo. Two entries may not name one remote |
| `paths:` | empty | checkouts on this machine. `~/` expands, `*`, `?`, and `[…]` match, and each path must be absolute. They identify a repo that has no origin, and they are the list `install-rules` and `doctor` walk |
| `projects:` | empty | the projects that apply inside a checkout. Each must exist under `projects:`. A project whose `path:` is inside the checkout applies as well, without being listed |
| `default:` | none | the project to pick when several apply. It must exist under `projects:` |

What the block changes:

- `skram run <target>` with no project name works inside a checkout: the
  project is the one that applies there, or `default:` when several do. With
  several and no default, the run is refused and the candidates are named.
  Inside a checkout other than the project's `path:` (a worktree, a second
  clone), the run happens in that checkout.
- The guard's `make`, `task`, `just`, `kubectl`, and `docker` rules, and the
  `allow:` patterns, use the projects that apply where the command runs.
- `skram agent install-rules` visits every path listed here, plus the git root
  of every project's `path:`, plus each of those checkouts' own linked
  worktrees, and writes the machine-local agent files there.
- `skram doctor` reports an entry whose remote no checkout on this machine
  matches, and a path whose spelling differs from the one on disk.

`repos.<name>.namespaces` is in the schema for Skram Vault, which reads it.
Skram passes it through untouched and never validates it.

## `logs_dir:` and `queue_dir:`

```yaml
logs_dir: ~/.local/share/skram/logs
queue_dir: ~/.local/share/skram/queue
```

Those are the defaults, with `$XDG_DATA_HOME/skram` in place of
`~/.local/share/skram` when that variable is set. Both directories are created
the first time they are used; until then `skram doctor` notes them as not
created yet, which is not an issue.

`logs_dir:` holds one directory per job, named with the job id, containing
`raw.log` (everything the job printed), `events.jsonl` (one JSON object per
line), and `status.json` (project, target, actor, exit code, timing). Job
history, and so every estimate, is read from these directories, and
`skram prune` removes the old ones.

`queue_dir:` holds `queue.json` (the items, the hold, the drain flag), the
lock file that guards it, `queue-history/` (archived queue files), and
`queue-processor.log`.

Point everything that shares a resource at one `queue_dir:`. Two settings mean
two queues, two processors, and two jobs on the same resource at once.

## Blocks another product owns

`vault:` and `tunnel:` belong to Skram Vault and Skram Tunnel. Skram reads
neither and complains about neither, so one file can configure all three
products.

## Keys that are now errors

These were removed before the first public release. Skram refuses to load a
file that still has one, rather than ignoring it, so an old file fails with a
message instead of a surprise.

| Key | What to do | Message |
| --- | --- | --- |
| `consumers:` | delete it | `config: 'consumers' is no longer supported; delete it (embedders are not checked by doctor any more)` |
| `cursor:` | delete it. Skram writes no editor-specific files into a checkout | `config: 'cursor' is no longer supported; skram writes no per-repo tool config (see docs/agent-setup.md in the skram repository)` |
| `projects.<name>.rules_dir` | list the checkout under `repos.<name>.paths` | `config: projects.my-app.rules_dir is no longer supported; list the checkout in repos.<name>.paths` |
| `projects.<name>.rules` | delete it. Skram installs no skills | `config: projects.my-app.rules is no longer supported; skram installs no skills (see docs/agent-setup.md in the skram repository)` |
| `projects.<name>.cursor` | delete it, like the top-level key | `config: projects.my-app.cursor is no longer supported; skram writes no per-repo tool config (see docs/agent-setup.md in the skram repository)` |

## Checking the file

```bash
skram doctor                    # projects, paths, targets, data directories
skram discover                  # every project with the targets it exposes
skram discover --json           # the same, with resources, inputs, and the ephemeral flag
skram discover -C ../my-app--feature   # read a worktree's own files for its repo's projects
```

`skram doctor` is the one to run after an edit. It reports a path that does
not exist, a backend file it cannot find, an `expose:` or `ephemeral:` name
that is not a target, a repo with no checkout here, an input with no default
directory here, and a path whose casing differs from disk.

Back to the [documentation index](README.md), or the
[README](../README.md) for installing Skram and the first loop.
