# KlipperScreen broken against Moonraker v0.11.0 (FLSUN V400 / Speeder Pad)

Date: 2026-09-20
Status: diagnosis confirmed from live logs. Fix NOT yet written, NOT yet tested.

## Symptom

On the Speeder Pad, KlipperScreen shows no live machine stats and an empty
file browser. Mainsail in the browser works normally. Restarting KlipperScreen
shows a single snapshot of stats, then nothing updates.

## Evidence

From `~/printer_data/logs/KlipperScreen.log`, 2026-09-20 10:15:

```
KlippyWebsocket.py:on_open()        - Moonraker Websocket Open
KlippyWebsocket.py:identify_client()- Sending server.connection.identify
screen.py:init_klipper()            - moonraker_version: v0.11.0-1-g1cfb0c4
KlippyWebsocket.py:object_subscription() - Sending printer.objects.subscribe
KlippyWebsocket.py:get_file_list()  - Sending server.files.list
files.py:_callback()                - {'code': -32602, 'message': 'Unauthorized'}
```

Configured printer, same log:

```
"moonraker_host": "127.0.0.1", "moonraker_port": "7125", "moonraker_api_key": ""
```

What this proves, line by line:

| Observation | Conclusion |
|---|---|
| REST `server.info` returns full component list | The HTTP/REST path authenticates fine |
| Websocket opens and `identify` is sent | TCP and handshake are fine |
| `server.files.list` returns `-32602 Unauthorized` | Moonraker rejects the websocket connection's credentials |
| Stats freeze after the first read | `printer.objects.subscribe` is refused by the same rejection |
| `moonraker_api_key` is `""` | No API key is configured on this machine |

Empty file browser and frozen stats are one fault, not two.

## Root cause

Moonraker v0.11.0 tightened credential validation. KlipperScreen on this
machine still sends an EMPTY credential in two places, and an empty credential
is now treated as an invalid one rather than as absent.

Repo on the machine: `Guilouz/KlipperScreen-Flsun-Speeder-Pad`, HEAD `4a447b6`,
last upstream commit 2024-08-29. Roughly two years stale.

| File | Current code | Problem |
|---|---|---|
| `ks_includes/KlippyWebsocket.py` line 56 | `self.ws_url = f"{self.ws_proto}://{self._url}/websocket?token={self.api_key}"` | With no key this sends `?token=` (empty). v0.11.0 rejects it. |
| `ks_includes/KlippyWebsocket.py`, `MoonrakerApi.identify_client` | `"api_key": f"{api_key}"` | With no key this sends `"api_key": ""`. v0.11.0 rejects it. |

The `x-api-key` header path is already correct: `KlippyWebsocket.__init__` sets
`self.header = {"x-api-key": api_key} if api_key else {}`, which correctly
sends nothing when there is no key.

## Upstream fix

Upstream repo: `KlipperScreen/KlipperScreen`. Issue #1760. Three commits:

| # | SHA | Date | Message |
|---|---|---|---|
| 1 | `f081f586e0aa6b92177c8982359089065e7dad90` | 2026-08-29 | refactor: use header-only API key auth for websocket connections #1760 |
| 2 | `27555e80e842fc34358a7ad65c90e081dcb5e3e8` | 2026-09-04 | fix: accept False for a blank api_key and cleanup handling fixes #1760 |
| 3 | `0ccec8bec1c3f320abf7e87b0c3307220a11bbba` | 2026-09-04 | refactor: send the agent for completeness #1760 |

What each one does:

1. Drops `?token={api_key}` from the websocket URL, and drops the `api_key`
   request param in `MoonrakerApi.py`.
2. Drops `api_key` from the `KlippyUDS`/`KlippyWebsocket` constructors and from
   the stored state, converts the literal string `"False"` to `""` in
   `screen.py`, and makes `identify` read the normalised value.
3. Adds `header=["User-Agent: KlipperScreen"]` to the websocket connection.

## Why the commits cannot be cherry-picked

The fork predates a large upstream refactor. Files the upstream patches touch
do not exist in the fork:

| Upstream file touched | Present in fork? |
|---|---|
| `ks_includes/MoonrakerApi.py` | No. The fork defines `class MoonrakerApi` inside `KlippyWebsocket.py` |
| `ks_includes/KlippyUDS.py` | No. Unix socket support postdates the fork |
| `ks_includes/KlippyRest.py` | Yes in the fork, renamed away upstream |

`git cherry-pick` would fail on every hunk. The change has to be hand-ported
and squashed into one commit, which is what was asked for anyway.

## Planned port (not yet applied)

Two files, three edits.

| File | Change |
|---|---|
| `ks_includes/KlippyWebsocket.py` line 56 | `self.ws_url = f"{self.ws_proto}://{self._url}/websocket"` |
| `ks_includes/KlippyWebsocket.py` `connect()` | add `User-Agent: KlipperScreen` to the header already being passed |
| `ks_includes/KlippyWebsocket.py` `identify_client` | omit the `api_key` field when there is no key |
| `screen.py` line ~696 | normalise the value passed to `identify_client`, so the string `"False"` becomes empty |

The `"False"` normalisation is NOT needed for this machine, whose key is
already `""`. It is included because the upstream fix covers it and because a
PR back to Guilouz should fix both shapes, not only the one seen here.

## Open items

| # | Item | State |
|---|---|---|
| 1 | Fork `Guilouz/KlipperScreen-Flsun-Speeder-Pad` to the user's GitHub | Not done. No `*Screen*` repo exists under `VLoorenDeJong` as of this check |
| 2 | Apply the port as one commit on the fork | Not started |
| 3 | Repoint the Pad: git remote AND `origin:` in `[update_manager KlipperScreen]` in `~/printer_data/config/moonraker.conf` | Not started |
| 4 | Update via the Mainsail update manager, which restarts the service | Not started |
| 5 | Verify stats and file browser | Not started |
| 6 | PR fork to `Guilouz/KlipperScreen-Flsun-Speeder-Pad`, referencing its issue #50 | Not started |

## Notes and risks

- Moonraker's update manager pins the repo in `moonraker.conf`. Changing only
  the git remote is not enough, `origin:` must change too or the update is
  refused.
- Moonraker marks a repo INVALID or DIRTY if it has local uncommitted edits.
  The workaround posted in issue #50 tells people to edit the files in place,
  which leaves the repo dirty. If that workaround was already applied on this
  machine, it must be reverted before repointing the remote.
- `gh` CLI is not installed on the Windows workstation, so GitHub actions such
  as forking and opening the PR have to be done in the browser, or `gh` has to
  be installed first.
- Guilouz's repo has an open issue for exactly this: #50, "KlipperScreen not
  updating real time data after moonraker update", opened 2026-08-26, with a
  confirmed in-place workaround by user `h3rm`.

## Verification status

- Diagnosis: confirmed from the live log quoted above.
- Fix: written down only. No code changed, nothing built, nothing run.
