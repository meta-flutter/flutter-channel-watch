# flutter-channel-watch

Watches the Flutter stable channel and dispatches the things that depend on it.

Two signals, checked about every 15 minutes:

| file | holds | drives |
|---|---|---|
| `stable` | `bin/internal/engine.version` from `flutter/flutter` | the flutter-engine builds |
| `stable-version` | `current_release.stable` resolved through `releases_linux.json` | the meta-flutter roll |
| `last-checked` | UTC timestamp of the last successful check | liveness |

They are separate because they are separate things: the engine SHA is the SRCREV
the engine builds take, while meta-flutter pins `FLUTTER_SDK_TAG` to a version
string resolved out of the release feed. The engine can move without a release
existing in that feed.

## Reading a run

**Green with no commit** means nothing changed. That is the normal outcome and it
is quiet on purpose. **Red means something is wrong** — a fetch failed, a fetched
value did not look like a SHA or a version, or a dispatch was rejected.

`last-checked` is committed once a day even when nothing has changed. That keeps
the schedule alive — GitHub disables scheduled workflows after 60 days without
repository activity, which is what silently stopped this watcher between
2026-04-16 and 2026-06-15 — and it doubles as the liveness signal. If
`last-checked` is more than a day or two old, the watch is not running.

## If it stops

A workflow disabled for inactivity does not resume on its own:

    gh workflow enable stable-channel-watch --repo meta-flutter/flutter-channel-watch

## Secrets

`WORKFLOW` — a token that can dispatch workflows in `meta-flutter/flutter-engine`.
Nothing else is needed; the state commits use the built-in `GITHUB_TOKEN`.
