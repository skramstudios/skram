# herdr

Skram ships a [herdr](https://herdr.dev) plugin. Inside a herdr session the
queue processor shows the running job in each workspace's sidebar row, and
the TUI opens as a popup.

## Install

Needs herdr 0.9 or newer, and `skram` on `PATH` — the plugin's commands run
`skram` by name (`skram doctor` checks):

```bash
mkdir -p ~/.config/skram/herdr-plugin
curl -fsSL -o ~/.config/skram/herdr-plugin/herdr-plugin.toml \
  https://raw.githubusercontent.com/skramstudios/skram/main/herdr-plugin/herdr-plugin.toml
herdr plugin link ~/.config/skram/herdr-plugin
```

The `skram herdr` command group only appears in `skram --help` once herdr is
installed; its subcommands are documented below whether or not it shows.

## Sidebar row

Add a `$skram` row to the space rows in `~/.config/herdr/config.toml`, then
reload:

```toml
[ui.sidebar.spaces]
rows = [
  ["state_icon", "workspace"],
  ["branch", "git_status"],
  [{ token = "$skram", fg = "#f9e2af" }],
]
```

```bash
herdr server reload-config
```

The row reads `build 1m20s/4m10s · 2 queued` while a job runs for the
project a workspace pane stands in, in the checkout that pane stands in (a
job in another worktree shows in that worktree's panes, named
`build (repo--T-7) …`), or that was enqueued from that workspace. It refreshes every 30 seconds and
disappears once the queue is idle.

```bash
skram herdr report --dry-run   # preview what the processor would send; sends nothing
```

`SKRAM_HERDR=0` turns reporting off.

## Popups and actions

Three popup panes, opened by the plugin or by hand:

- `skram.open-tui` — the TUI (`skram tui`)
- `skram.status` — `skram status`
- `skram.logs` — `skram logs -F`

Two actions that run something and show the result as a herdr notification:

- `skram.kill-current` — kill the running job
- `skram.queue-drain` — let running jobs finish, then stop the processor

Any of the five run without a binding, through `herdr plugin action invoke
<id>`.

## Key bindings

`prefix+s` is herdr's own `settings` binding, so `skram.open-tui` needs a
free one:

```toml
[[keys.command]]
key = "prefix+shift+s"
type = "plugin_action"
command = "skram.open-tui"
description = "skram TUI"
```

Kill and drain earn one each:

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

`prefix+alt+s` → `skram.status` and `prefix+alt+l` → `skram.logs`, bound the
same way, complete the set.

herdr has no binding type that opens a whole tab, so a persistent skram tab
is a one-time layout: `prefix+c`, rename it, split it, and run `skram tui`
in one pane and `skram logs -F` in the other.

## When a workspace opens

A new workspace with skram projects in it gets a one-time notification
suggesting a binding for `skram.open-tui`, so the hint appears once per
workspace rather than every session.

## `skram herdr` subcommands

```bash
skram herdr report --dry-run      # one pass of the sidebar report; --dry-run sends nothing
skram herdr open tui              # open a popup pane in the current session: tui, status, or logs
skram herdr pane status           # the status popup's body; used by the plugin manifest, not run by hand
skram herdr action kill-current   # run a plugin action; kill-current, queue-drain, or hint; used by the manifest and workspace.created
```

`pane` and `action` are how the plugin manifest and the `workspace.created`
event invoke skram — they are not meant to be typed directly, but nothing
stops you from running them the same way the plugin does.

## What `skram doctor` reports

```
[ok] herdr: /opt/homebrew/bin/herdr (2 running session(s); the processor reports the running job per workspace)
```

or, when reporting is off or herdr is unavailable:

- `[--] herdr: reporting disabled (SKRAM_HERDR=0)`
- `[--] herdr: not installed (sidebar rows and the popup TUI need herdr 0.9+)`
- `[--] herdr: installed, no running session`
- `[!!] herdr: skram not on PATH; the plugin's popup and actions run "skram" by name`
