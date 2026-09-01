# flutter-channel-watch: reliability and the meta-flutter auto-roll

Tracking: meta-flutter/flutter-channel-watch#4, meta-flutter/meta-flutter#864

## Where it stands

The watch is off, and has been since 2026-06-15.

```
$ gh api repos/meta-flutter/flutter-channel-watch/actions/workflows
  "state": "disabled_inactivity"
```

GitHub disables scheduled workflows after 60 days without repository
activity. Last commit 2026-04-16, last run 2026-06-15 — 04-16 plus 60 days,
to the day.

| | |
|---|---|
| `stable` in this repo | `59aa584f…` |
| live `engine.version` | `a804b261…` |
| flutter stable release | `d3b14c87…` (3.47.2, 2026-08-27) |
| meta-flutter pins | 3.47.1 |

Four and a half months of engine rolls went undispatched, and the layer is a
release behind.

**The design is why nobody noticed.** The no-change path is
`echo "No changes" | false` — a deliberate job failure — so a healthy watcher
is red on nearly every run. The last 100 runs are 100 failures. A dead watcher
and a working one are indistinguishable. `Clean Failed Runs` then deletes the
evidence; its filter is `head_branch != "master"` on a repo whose default
branch is `main`, so it matches everything, real failures included. The README
documents the result as a feature.

## What it should do

One detection, two consumers:

```
poll (every 15 min)
  ├─ engine.version changed   → dispatch flutter-engine x86_64 / arm64 / armv7hf
  └─ stable version changed   → repository_dispatch → meta-flutter roll
                                  → tools/roll_meta_flutter.py --version=$V
                                  → branch auto/roll-$V, draft PR, CI gates it
```

Two signals, because they are two different things. `engine.version` is the
engine SRCREV the flutter-engine builds take. meta-flutter pins
`FLUTTER_SDK_TAG` to a *version* string (`3.47.1`) resolved out of
`releases_linux.json`. The engine can move without a release existing in the
feed the layer reads, and a roll dispatched on the engine SHA would have no
version to pin.

Never auto-merge. The PR is the deliverable; CI is the gate; a human merges.

## State in the repo

| file | holds | written |
|---|---|---|
| `stable` | engine.version SHA | on change (keep the name; flutter-engine consumers read it) |
| `stable-version` | stable release version, e.g. `3.47.2` | on change |
| `last-checked` | UTC timestamp of the last poll | once per UTC day |

`last-checked` is the fix for the inactivity disable *and* the liveness signal:
if it is more than a day or two old, the watch is dead. One commit per day is
enough activity to hold off the 60-day cutoff and quiet enough to ignore.

## Work

### 1. Invert the exit convention — do this first

No-change is a green no-op. Set step outputs and gate later steps on `if:`.
Nothing else on this list is observable until this lands, because today a
failure and a success look the same.

```yaml
- id: check
  run: |
    echo "engine_changed=..." >> "$GITHUB_OUTPUT"
    echo "version_changed=..." >> "$GITHUB_OUTPUT"
- if: steps.check.outputs.engine_changed == 'true'
  ...
```

### 2. Delete `Clean Failed Runs`

It exists only to hide the self-inflicted red, deletes genuine failures along
with it, and spends a PAT doing so. Once (1) lands there is nothing to clean.

### 3. Fetch safely

```sh
curl --fail --show-error --silent --retry 3 --retry-delay 5 \
     --max-time 30 -o engine.new "$ENGINE_URL"
grep -Eq '^[0-9a-f]{40}$' engine.new || { echo "::error::bad engine.version"; exit 1; }
```

`wget -O stable <url>` truncates the destination before it knows whether the
request succeeded. A 404 or a storage blip leaves `stable` empty or holding an
HTML error page, which reads as a change and dispatches builds with a garbage
`srcrev`. Validate before moving into place, never write the state file
directly from the network.

Same treatment for the release feed: parse `current_release.stable`, look up
its `version`, and assert it matches `^[0-9]+\.[0-9]+\.[0-9]+$` before use.

### 4. Check every dispatch

The three `curl -X POST` calls ignore their status. An expired PAT returns 401
and the run still goes green. Use `--fail-with-body` and fail the step.

### 5. Remove the dead ssh block

```sh
echo ${{ secrets.SSH_KNOWN_HOSTS }} | >> $HOME/.ssh/known_hosts
```

pipes into a bare redirect and writes nothing. The push works over the
`actions/checkout` token regardless, so the entire ssh-agent setup is inert.
The unquoted secret interpolation is also an injection surface. Delete it.

### 6. Daily heartbeat

Commit `last-checked` when the UTC date changes. Keeps the cron alive and
makes staleness visible.

### 7. Dispatch the meta-flutter roll

New, gated on `version_changed`:

```sh
gh api repos/meta-flutter/meta-flutter/dispatches \
  -f event_type=flutter-stable-roll \
  -F "client_payload[version]=$VERSION"
```

### 8. Guard against overlap

`concurrency: { group: watch, cancel-in-progress: false }`. A 15-minute cron
with a slow API call can otherwise overlap itself and double-dispatch.

### 9. Credentials

`USER`, `API_TOKEN`, `WORKFLOW`, `SSH_PRIVATE_KEY` today. (5) removes the ssh
key and (2) removes the `USER`/`API_TOKEN` pair, leaving one token for
dispatch. Prefer a GitHub App installation token; a fine-grained PAT with a
calendar reminder otherwise. After (1) and (4), expiry at least turns the run
red instead of passing silently.

Items 1, 3, and 6 are the ones that would have caught the outage.

## On the meta-flutter side

Tracked in meta-flutter#864. Blocking prerequisite: **meta-flutter#863** —
`roll_meta_flutter.py` submits its per-app work to a `ThreadPoolExecutor` and
never reads the futures, so a failed clone or a license mismatch is captured
and discarded and the script exits 0. Unattended rolling is not safe until the
roll can fail.

Also open there: which branch rolls first (convention is wrynose, but the
tooling and the richest CI are on master), and how the roll job gets a Flutter
SDK at the version it has only just determined.

## Alternative considered

Move the watch into meta-flutter and retire this repo. meta-flutter takes
commits and CI daily, so its schedules never approach the inactivity cutoff —
that removes the failure mode rather than patching it, and item 6 becomes
unnecessary.

Not chosen, because `stable` here is a published signal other consumers read,
and the flutter-engine dispatches belong next to it. Worth revisiting if the
heartbeat proves fragile: a trigger should not depend on a repo whose only
activity is the trigger itself.

## Order

1 → 2 → 3 → 6 first: that is the outage, and it is self-contained.
Then 4, 5, 8, 9 as cleanup.
7 last, once meta-flutter#863 has landed and the roll workflow exists.

Re-enable the workflow by hand after the first commit lands
(`gh workflow enable`); a repo disabled for inactivity does not resume on its
own.
