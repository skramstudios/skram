# skram

**The job queue you and your coding agents share.**

You have one KinD cluster, one Docker VM, one shared build host. You have three
repos, a couple of Claude Code sessions, and yourself, all wanting to build,
deploy, and test against it at the same time. skram serializes that work,
remembers every run, and tells the agents what happened, so nobody clobbers
anybody.

```
$ skram run apps cluster-run --json
{ "item_id": "q_e981b374", "estimate_seconds": 348, "wait": "skram wait q_e981b374" }

$ skram wait q_e981b374
apps/cluster-run J20260905_205549  [completed] exit=0 (5m42s)
```

skram is a single Go binary, no server, no database. A job is three files on
disk. The queue is a JSON file under `flock`. That is the whole runtime.

## What it does

- **One queue, many callers.** Every `skram run` from any repo, shell, or
  agent lands in the same queue: one job at a time per shared resource,
  lanes in parallel. Jobs run in their own process group. `skram kill`
  kills the whole group.
- **Humans attach, agents wait.** `-f` enqueues *and* attaches: block,
  stream, exit with the job's code. `skram wait <item>` does the same without
  the stream, for an agent that enqueued with `--json`. `--now` bypasses the
  queue, but only for targets you mark `ephemeral` (read-only things like
  `status`).
- **Every run is remembered.** `~/.local/share/skram/logs/<job>/` holds
  `raw.log`, `events.jsonl`, and `status.json`. Secrets are redacted on the
  way in. `skram prune` reclaims old ones.
- **Estimates from history.** Enqueue prints how long this command usually
  takes and when the queue clears. `status` shows time remaining per running
  job. Keys are hierarchical, so a new flag combination inherits its target's
  history.
- **Machine-readable everywhere.** `--json` on `run`, `status`, `queue list`,
  `logs` (streams `events.jsonl`), `discover` (targets, help, ephemeral flag,
  estimate), and `wait`.
- **Failure policy.** `--on-error stop` cancels the same project's pending
  jobs after a failure. Exit codes are classified semantically (a `make`
  "nothing to be done" is not a failure).
- **Notifications** on job end, through herdr inside a session or macOS
  otherwise. **Dashboard** with SSE log streaming, optional ngrok tunnel and
  QR code. **Prometheus** exporter.
- **Docker disk reaper.** After a build job, old timestamped image tags and
  build cache beyond a budget are removed so the Docker VM never fills and
  takes your cluster's volumes with it.

## Install

Releases are built for macOS and Linux, on amd64 and arm64.

```bash
curl -fsSL https://raw.githubusercontent.com/skramstudios/skram/main/install.sh | sh
skram version
```

The script downloads the latest release for your machine, verifies it against
the release's `checksums.txt`, and puts `skram` in `~/.local/bin`, which must
be on your `PATH`. `VERSION=v0.13.0` picks a release and `BIN_DIR=…` a
different directory. To do it by hand, take the archive for your platform
from the [releases page](https://github.com/skramstudios/skram/releases),
check it against `checksums.txt`, and unpack `skram` from it; shell
completions for bash, zsh, and fish are in the archive's `completions/`.

If you downloaded the archive in a browser, macOS quarantines the binary and
refuses to run it. Clear the flag once:

```bash
xattr -d com.apple.quarantine ~/.local/bin/skram
```

## Quickstart

skram dispatches to a shell script with a `case` statement per target. That
is deliberately the lowest common denominator: it wraps a Makefile, a
Taskfile, `kubectl`, or anything else without skram having to understand it.

**1. Give a project an ops script.**

```bash
#!/usr/bin/env bash
# ops.sh — one case arm per target. Indented "name  help" lines become help text.
usage() {
  cat <<EOF
Usage: ops.sh <target>
  status          Show cluster status
  cluster-run     Create the KinD cluster and deploy everything
  test            Run the e2e suite
EOF
}
case "$1" in
  status)
    kubectl get pods -A ;;
  cluster-run)
    make cluster && make deploy ;;
  test)
    shift; go test ./e2e/... "$@" ;;
  *)
    usage; exit 2 ;;
esac
```

**2. Register it.**

```bash
cd ~/dev/apps && skram init --detect         # finds ops.sh / Makefile / Taskfile / justfile, writes the entry below on confirm
```

```yaml
projects:
  apps:
    path: /home/you/dev/apps
    script: ops.sh           # or makefile: Makefile, taskfile: Taskfile.yml, justfile: justfile
    resource: kind-dev       # optional; projects sharing a resource take turns
    ephemeral: [status]      # add by hand: read-only targets run immediately, no queue, no history
```

`init --detect` proposes the first entry point it finds (a `case` script,
then a Makefile, a Taskfile, a justfile; `--backend` picks another,
`--name` renames, `--yes` skips the prompt, `--dry-run` and `--json` never
write). Targets are the script's `case` arms, a Makefile's `.PHONY` names and
`target: ## help` rules, a Taskfile's tasks (`desc:` is the help), or a
justfile's doc-commented recipes; includes are not followed.

**3. Use it.**

```bash
skram discover                               # every project, target, help, and typical duration
skram run apps cluster-run                   # enqueue; prints the estimate
skram run cluster-run                        # inside the repo the project name is optional
skram run apps test --scope=auth             # everything after the target goes to the script
skram status                                 # running job, time remaining, queue ETA
skram logs -F                                # follow the running job
skram run apps status --now                  # ephemeral: runs right now, no job dir
skram run apps test -f                       # attach: block, stream, exit with its code
skram kill                                   # stop the running job (whole process group)
```

## The agent loop

An agent should never drive the ops script directly. The loop is enqueue,
wait, read:

```bash
skram run apps test --json                   # → item_id, estimate, "wait": "skram wait q_…"
skram wait q_e981b374 --json --timeout 30m   # blocks; exit code is the job's (125 cancelled, 137 killed, 124 timeout)
skram logs q_e981b374 --json --tail 40       # events.jsonl, one JSON object per line
skram status --json                          # what is running, what is queued, when it clears
```

`skram wait --queue` blocks until nothing is pending or running and fails if
anything that was in flight failed. That replaces the "poll `status` in a
loop" pattern that used to make up half of all job history.

`wait` exit codes:

| Code | Meaning |
|------|---------|
| 0 | the job completed |
| the job's own | it ran and failed |
| 124 | `--timeout` elapsed; the job is still going |
| 125 | the item was cancelled before it ran (`cancelled_by` says by whom) |
| 137 | the job was killed |

`skram queue cancel <item>` withdraws a pending item you no longer need.

The same loop is available over MCP: `skram mcp` serves `discover`, `run`,
`wait`, `explain`, `logs`, and `status` as tools with identical JSON, carries
the rules as the server's instructions, and exposes each project's target
table as a resource. `skram agent install-rules` writes the same text into
every checkout the `repos:` block names, as a machine-local `AGENTS.local.md`
(imported by `CLAUDE.local.md`, both listed in `.git/info/exclude`) — nothing
committed, so it works in a repo you cannot touch. A repo is its upstream
(`remote: org/name`), so two checkouts of it get the same files. Register the
server once at user scope: `claude mcp add --scope user skram -- skram mcp`.

The knowledge vault — a git-backed markdown store where agents keep what the
code doesn't say (contract quirks, races, debugging dead ends, decisions) —
now lives in its own tool, [`skram-vault`](https://github.com/skramstudios/skram-vault).
It runs beside `skram` against a vault you own, with its own CLI, MCP server,
and `AGENTS.local.md` section. `skram lore|spec|ticket|vault` here are
one-line redirects that point you at the `skram-vault` equivalent; a `vault:`
block in a shared config is ignored by `skram` and read by `skram-vault`.

## Sharing a resource with your agents

The setup this was built for: one [herdr](https://herdr.dev) session per
repo, Claude Code in each, all deploying to the same local cluster.

- Agents enqueue and `wait`. You enqueue with `-f` when you want to watch,
  or plain `run` and go back to what you were doing.
- Each project is a lane named after its shared resource (`resource:` in
  the config; default is the project name). Lanes run in parallel and
  serialise within themselves, so `build-host` never waits behind a `kind-dev`
  cluster rebuild, while two projects that both say `resource: kind-dev`
  still take turns. A project that touches two clusters lists both:
  `resource: [kind-dev, kind-staging]`. `skram wait --resource kind-dev` and
  `skram kill --resource kind-dev` address one lane.
- Nobody bypasses the queue for anything that mutates the cluster. The only
  way around it is `--now`, and that only works for targets you have listed
  as `ephemeral`.
- `--on-error stop` on a cluster rebuild cancels the queued jobs in that lane
  (every project sharing the resource) if the rebuild fails, without
  touching another lane's jobs.
- Job end fires a herdr notification (or a macOS one outside herdr).

### Notifications on macOS

Queued jobs notify on completion: through herdr inside a session, otherwise
through macOS. macOS only shows them once `terminal-notifier` is installed
and registered as a GUI app (the `osascript` fallback is usually silent).
One-time setup per machine:

```bash
brew install terminal-notifier
cp -R /opt/homebrew/opt/terminal-notifier/terminal-notifier.app /Applications/
open /Applications/terminal-notifier.app                # registers it with LaunchServices
terminal-notifier -title "Test" -message "Hello World"  # triggers the permission prompt; allow it
skram doctor                                            # [ok] notify: terminal-notifier
```

`skram doctor` reports the notifier in use; `SKRAM_NOTIFY=0` disables
notifications.

## herdr

skram ships a [herdr](https://herdr.dev) plugin. Inside a herdr session the
queue processor shows the running job in each workspace's sidebar row, and
the TUI opens as a popup.

Install the plugin (needs herdr 0.9 or newer and `skram` on PATH):

```bash
mkdir -p ~/.config/skram/herdr-plugin
curl -fsSL -o ~/.config/skram/herdr-plugin/herdr-plugin.toml \
  https://raw.githubusercontent.com/skramstudios/skram/main/herdr-plugin/herdr-plugin.toml
herdr plugin link ~/.config/skram/herdr-plugin
```

Show the running job per workspace. Add a `$skram` row to the space rows
in `~/.config/herdr/config.toml`, then `herdr server reload-config`:

```toml
[ui.sidebar.spaces]
rows = [
  ["state_icon", "workspace"],
  ["branch", "git_status"],
  [{ token = "$skram", fg = "#f9e2af" }],
]
```

The row reads `build 1m20s/4m10s · 2 queued` while a job runs for a
project whose path (or git root) contains one of the workspace's pane
directories, or that was enqueued from that workspace. It refreshes every
30 s and disappears when the queue is idle. `skram herdr report --dry-run`
prints what the processor would send; `SKRAM_HERDR=0` turns reporting off.

Open the TUI with a key. `prefix+s` is herdr's `settings` binding, so pick
a free one:

```toml
[[keys.command]]
key = "prefix+shift+s"
type = "plugin_action"
command = "skram.open-tui"
description = "skram TUI"
```

The other actions are `skram.status`, `skram.logs` (popups),
`skram.kill-current`, and `skram.queue-drain` (run and notify), all
available through `herdr plugin action invoke <id>` or a binding of their
own. Kill and drain earn one each:

```toml
[[keys.command]]
key = "prefix+alt+k"
type = "plugin_action"
command = "skram.kill-current"
description = "skram: kill current job"

[[keys.command]]
key = "prefix+alt+d"
type = "plugin_action"
command = "skram.queue-drain"
description = "skram: drain queue"
```

`prefix+alt+s` → `skram.status` and `prefix+alt+l` → `skram.logs`, bound
the same way, complete the set. herdr has no binding type that opens a whole tab, so
a persistent skram tab is a one-time layout: `prefix+c`, rename, split, run
`skram tui` in one pane and `skram logs -F` in the other. `skram doctor` reports the herdr state (`[ok] herdr: … (N running
session(s))`).

## Sharing what you just built

`skram dashboard` shares skram's own monitoring UI, optionally over an ngrok
tunnel with a QR code.

Sharing the **app itself** — including the login in front of it, which a
plain `ngrok http 3000` cannot do — moved to its own tool:
`skram-tunnel` (release coming). Install it
separately and give it its own `tunnel:` config block; `skram tunnel <args>`
now just points you at it.

## What it is not

skram is not a CI system, not a durable workflow engine, and not a
per-project task runner. It does not do parallel groups, remote execution,
or input-hash caching (yet, or ever, respectively).

If you only need a shell job queue for one terminal, use
[pueue](https://github.com/Nukesor/pueue). skram is for when several
callers, some of them not human, share one thing.

## More

- `CHANGELOG.md`: what changed in each release.
- [config.md](https://github.com/skramstudios/.github/blob/main/config.md):
  the one config file the Skram tools share.

## License

Apache-2.0. See `LICENSE` and `NOTICE`; `THIRD_PARTY_NOTICES.md` carries the
licences of the Go modules built into the binary.
