# Notifications

Queued jobs notify when they finish: through herdr inside a session,
otherwise through a macOS notification. `SKRAM_NOTIFY=0` disables
notifications entirely.

## macOS setup

macOS only shows completion notifications once `terminal-notifier` is
installed and registered as a GUI app; the `osascript` fallback macOS falls
back to is usually silent. One-time setup per machine:

```bash
brew install terminal-notifier
cp -R /opt/homebrew/opt/terminal-notifier/terminal-notifier.app /Applications/
open /Applications/terminal-notifier.app                # registers it with LaunchServices
terminal-notifier -title "Test" -message "Hello World"  # triggers the permission prompt; allow it
skram doctor                                            # [ok] notify: terminal-notifier
```

## What `skram doctor` reports

- `[ok] notify: terminal-notifier (queued jobs notify on completion; herdr inside a session)`
- `[--] notify: terminal-notifier not on PATH; completion notifications use osascript, which macOS may not show`
- `[--] notify: disabled (SKRAM_NOTIFY=0); queued jobs will not notify on completion`
- `[--] notify: no notifier on <os> (herdr inside a session only)` — Linux has no notifier backend yet

## Turning it off

`SKRAM_NOTIFY=0` disables notifications for queued jobs, both the herdr and
the macOS path. Nothing else `skram doctor` reports is affected.
