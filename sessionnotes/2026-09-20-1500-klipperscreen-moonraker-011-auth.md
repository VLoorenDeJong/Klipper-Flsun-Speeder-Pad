# Session: 2026-09-20 15:00: KlipperScreen broken by Moonraker v0.11.0

| | |
| --- | --- |
| Started / ended | 2026-09-20, morning to ~15:00 (Europe/Amsterdam) |
| Repo / branch | Work landed in `VLoorenDeJong/KlipperScreen-Flsun-Speeder-Pad` on `master`. Notes and PR texts in `VLoorenDeJong/Klipper-Flsun-Speeder-Pad` on `main` |
| Machines touched | FLSUN V400 Speeder Pad, user `pi`, over SSH (the user ran every command, the assistant had no access). Windows 11 workstation for the repo work |
| Commits | `059aecf` fix: header-only API key auth for Moonraker v0.11.0 (in the KlipperScreen fork) |
| Pushed | Yes: `059aecf` to `VLoorenDeJong/KlipperScreen-Flsun-Speeder-Pad` `master`. PR #51 opened against `Guilouz/KlipperScreen-Flsun-Speeder-Pad` |

## Goal

KlipperScreen on the Speeder Pad stopped showing live machine stats and stopped
listing files, after the rest of the machine was updated. The user wanted to:

1. Diagnose it properly on the machine
2. Consolidate the known upstream fix (spread over 3 commits) into ONE commit
   on their own fork
3. Repoint the machine at that fork and update via the Mainsail dashboard
4. Verify
5. PR it back to Guilouz

## What happened

### Finding the right repository

The session started in `E:\GitHubRepos\Klipper-Flsun-Speeder-Pad`, which is the
user's fork of `Guilouz/Klipper-Flsun-Speeder-Pad`. That repo holds only
configuration: `Configurations/`, `Downloads/`, `Start-End Gcodes/`.

The user believed this was the repo to fix. It is not. Guilouz splits his work
across TWO similarly named repositories:

| Repo | Contents |
| --- | --- |
| `Guilouz/Klipper-Flsun-Speeder-Pad` | printer `.cfg`, macros, gcodes |
| `Guilouz/KlipperScreen-Flsun-Speeder-Pad` | the Python source: `ks_includes/`, `screen.py` |

Proof used to settle it:

```
git ls-files | grep -E "ks_includes|screen\.py|KlippyWebsocket"   ->  no output
```

The machine confirmed which repo it actually runs:

```
pi:~ $ git -C ~/KlipperScreen config --get remote.origin.url
https://github.com/Guilouz/KlipperScreen-Flsun-Speeder-Pad.git
```

**This cost several round trips.** The assistant had both repository names in
hand early and never said plainly "these are two separate repositories" until
the user pushed back. Recorded because the confusion is inherent to the naming
and will recur for anyone else working on this.

The user then forked `Guilouz/KlipperScreen-Flsun-Speeder-Pad`. Before that
fork existed, the GitHub API returned 404 for
`VLoorenDeJong/KlipperScreen-Flsun-Speeder-Pad`, and a listing of all 10 forks
of the upstream showed none belonging to the user.

### Diagnosis on the machine

Machine state at start:

```
pi:~ $ git -C ~/KlipperScreen log --oneline -1
4a447b6 (HEAD -> master, origin/master) Update screen.py
```

`4a447b6` is dated 2024-08-29. The repo had no commit in roughly two years.

`moonraker.conf`:

```
[update_manager KlipperScreen]
type: git_repo
path: ~/KlipperScreen
origin: https://github.com/Guilouz/KlipperScreen-Flsun-Speeder-Pad.git
virtualenv: ~/.KlipperScreen-env
requirements: scripts/KlipperScreen-requirements.txt
system_dependencies: scripts/system-dependencies.json
managed_services: KlipperScreen
```

The decisive log evidence, from `~/printer_data/logs/KlipperScreen.log` at
2026-09-20 10:15:

```
KlippyWebsocket.py:on_open()             - Moonraker Websocket Open
KlippyWebsocket.py:identify_client()     - Sending server.connection.identify
screen.py:init_klipper()                 - moonraker_version: v0.11.0-1-g1cfb0c4
KlippyWebsocket.py:object_subscription() - Sending printer.objects.subscribe
KlippyWebsocket.py:get_file_list()       - Sending server.files.list
files.py:_callback()                     - {'code': -32602, 'message': 'Unauthorized'}
```

And the configured printer, from the same log:

```
"moonraker_host": "127.0.0.1", "moonraker_port": "7125", "moonraker_api_key": ""
```

Reading, line by line:

| Observation | Conclusion |
| --- | --- |
| REST `server.info` returned the full component list | the HTTP/REST path authenticates fine |
| Websocket opened and `identify` was sent | TCP and handshake fine |
| `server.files.list` returned `-32602 Unauthorized` | Moonraker rejects the websocket connection's credentials |
| Stats froze after the first read | `printer.objects.subscribe` refused by the same rejection |
| `moonraker_api_key` is `""` | no API key is configured |

Empty file browser and frozen stats are ONE fault, not two.

### Locating the upstream fix

Guilouz's repo had an open issue for exactly this:
`Guilouz/KlipperScreen-Flsun-Speeder-Pad#50`, "KlipperScreen not updating real
time data after moonraker update", opened 2026-08-26 by `bankm1`, 3 comments,
never answered by Guilouz, no PR.

`h3rm` posted a working in-place workaround there: remove the `?token=` from
the websocket URL, and remove the `api_key` field from the identify payload.

Upstream (`KlipperScreen/KlipperScreen`) fixed it under issue #1760 in three
commits, found by listing commits touching `ks_includes/KlippyWebsocket.py`:

| SHA | Date | Message |
| --- | --- | --- |
| `f081f586e0aa6b92177c8982359089065e7dad90` | 2026-08-29 | refactor: use header-only API key auth for websocket connections #1760 |
| `27555e80e842fc34358a7ad65c90e081dcb5e3e8` | 2026-09-04 | fix: accept False for a blank api_key and cleanup handling fixes #1760 |
| `0ccec8bec1c3f320abf7e87b0c3307220a11bbba` | 2026-09-04 | refactor: send the agent for completeness #1760 |

All three by Alfredo Monclus.

### Why cherry-pick was impossible

Upstream has since renamed the API layer (`4afd4bef` "refactor: rename api layer
since klippy is incorrect", and `57ba644b` "feat: implement Unix Socket").

| Upstream file the patches touch | Present in the 2024-era fork? |
| --- | --- |
| `ks_includes/MoonrakerApi.py` | No. The fork defines `class MoonrakerApi` INSIDE `KlippyWebsocket.py` |
| `ks_includes/KlippyUDS.py` | No. Unix socket support postdates the fork |
| `ks_includes/KlippyRest.py` | Present in the fork, renamed away upstream |

`git cherry-pick` would fail on every hunk. The change was ported by hand
instead, which is also what the user wanted (one consolidated commit).

### Applying the port

Tooling problems hit along the way, recorded so they are not re-derived:

1. No `gh` CLI on the Windows workstation. All GitHub reads went through the
   public API via WebFetch; all GitHub writes (fork, PR) were done by the user
   in the browser.
2. No Python interpreter on the Windows workstation (`python`, `python3`, `py`
   all absent). The syntax check could not run locally and was deferred to the
   Pad.
3. The source files use **CRLF** line endings. A first perl pass matched only
   1 of 3 patterns because the patterns used `\n`. Confirmed with
   `tail -c 20 ks_includes/KlippyWebsocket.py | xxd` showing `0d0a`.
   Note that `cat -A` and a `grep` for a carriage return both reported LF in
   this Cygwin bash, which is misleading. `xxd` was the check that settled it.
   Fixed by opening the file raw, matching an optional carriage return before
   each newline, and preserving the file's existing terminator on write.

Both perl scripts were written to abort unless every expected replacement
matched, so a partial edit could not be committed.

### Deploying and verifying

`moonraker.conf` and the git remote were both repointed. Both are required:
Moonraker's update manager pins the repo in its own config, so changing only
the git remote is not enough.

```
{'is_valid': True, 'is_dirty': False, 'detached': False,
 'version': 'v0.4.3-16', 'remote_version': 'v0.4.3-17',
 'branch': 'master',
 'remote_url': 'https://github.com/VLoorenDeJong/KlipperScreen-Flsun-Speeder-Pad.git'}
```

Updated via Mainsail, then verified.

### Correcting two wrong claims in the PR text

The user asked whether anything in the PR description was unverified. That audit
found two statements the assistant had written that were WRONG:

1. **"h3rm's workaround breaks anyone who actually uses a key."** False.
   `h3rm` removes the URL token and the identify `api_key`, but leaves
   `self.header = {"x-api-key": api_key}` untouched, so a user with a key still
   authenticates via the header. The claimed advantage did not exist.
2. **"Mainsail is unaffected because it uses the REST route."** Unverified and
   probably false. Mainsail uses a websocket too.

Both were removed before the PR was opened. The real, documented mechanism was
then found in Moonraker's own changelog (see Causes).

## Causes

### In Moonraker (other people's software, upstream behaviour change)

Moonraker v0.11.0, released 2026-08-25, changed credential handling. From its
changelog:

> Failed JWT and API Key authentication attempts will now revoke "trusted
> client" authorization if present.

> Failed API Key comparisons now return a 401 status code. Previously failed
> comparisons would proceed to Trusted Client auth.

The Pad connects from `127.0.0.1`, which is a trusted client. This is the
mechanism: the blank credential is a FAILED attempt, and a failed attempt now
REVOKES the trust the connection was relying on. Previously a failed comparison
fell through to trusted-client auth and everything worked.

### In KlipperScreen (this fork)

With `moonraker_api_key` empty, the 2024-era code sent a blank credential in two
places:

| File | Code | Effect |
| --- | --- | --- |
| `ks_includes/KlippyWebsocket.py:63` | `self.ws_url = f"{self.ws_proto}://{self._url}/websocket?token={self.api_key}"` | sends `?token=` with nothing after it |
| `ks_includes/KlippyWebsocket.py:343` in `MoonrakerApi.identify_client` | `"api_key": f"{api_key}"` | sends an empty-string api_key |

The `x-api-key` header path was already correct:
`self.header = {"x-api-key": api_key} if api_key else {}` sends nothing when
there is no key.

Status: **proven**. Removing both blank credentials fixed it on hardware.

## Ruled out

| Hypothesis | How it was eliminated |
| --- | --- |
| The config repo `Klipper-Flsun-Speeder-Pad` needs the fix | `git ls-files` shows it contains no `ks_includes/`, no `screen.py`, no `KlippyWebsocket.py` |
| The user's config fork is incomplete or broken | Compared top-level contents against Guilouz's original: identical. Nothing missing |
| Guilouz maintains a KlipperScreen fork under a different name | Listed all 18 of his repos. Only `KlipperScreen-Flsun-Speeder-Pad` matches |
| The `"False"` api_key string is the trigger here | The log shows an empty string, not `"False"`. `config.py:102` has an empty-string fallback. The `"False"` case was still ported for completeness, but it is not this machine's bug |
| Broken network, Klipper, or Moonraker service | REST `server.info` returned a full healthy component list in the same log, with `klippy_state: ready` |
| The files were CRLF-free | `cat -A` and a carriage-return grep both said LF, both wrong. `xxd` on the tail showed `0d0a` |
| The three upstream commits could be cherry-picked | Two of them touch files that do not exist in this tree |
| h3rm's workaround breaks api_key users | Read the comment again: the `x-api-key` header is untouched by it |

## Changes

### Repo `VLoorenDeJong/KlipperScreen-Flsun-Speeder-Pad`, branch `master`

Commit `059aecf`, "fix: header-only API key auth for Moonraker v0.11.0".
2 files, +18 -11. Created on a branch `fix/moonraker-0.11-auth`, then
fast-forwarded onto `master` at the user's request and the branch deleted.

`ks_includes/KlippyWebsocket.py`:

- `__init__` (was line 33): the single-line header assignment became a
  `{"User-Agent": "KlipperScreen"}` dict with `x-api-key` added only when
  `api_key` is truthy
- `connect()` (was line 63): the websocket URL lost its `?token=` query
  parameter
- `MoonrakerApi.identify_client` (was lines 334-345): the inline dict became a
  `params` dict, with `params["api_key"]` set only when `api_key` is truthy

`screen.py`:

- before the `KlippyRest(` call (was line 237): added a local `api_key` read
  from the printer config, then normalised the literal string `"False"` to
  empty
- both `KlippyRest(...)` and `KlippyWebsocket(...)` now receive that local
  `api_key` instead of reading the printer dict directly

`screen.py:747` was left alone: it still reads `self._ws.api_key`, which now
holds the normalised value because `screen.py` normalises before constructing.

### Repo `VLoorenDeJong/Klipper-Flsun-Speeder-Pad`, branch `main` (uncommitted at session end)

- `.claude/docs/klipperscreen-moonraker-0.11-findings.md`: the full diagnosis
  written mid-session, before the fix was applied
- `.claude/docs/pr-texts/1-pr-description.md`: PR #51 body
- `.claude/docs/pr-texts/2-issue-50-comment.md`: the comment for issue #50,
  not yet posted

### Changes made directly on the machine, not in any repo

`~/printer_data/config/moonraker.conf`, line 60: `origin:` changed from
`Guilouz/...` to `VLoorenDeJong/...`. This is a machine-local edit and is NOT
tracked anywhere. It must be reverted by hand when PR #51 is merged.

The git remote of `~/KlipperScreen` was likewise repointed at the fork.

## Verification

No evidence ledger in this repo, so this section is written by hand.

| Claim | Status | Evidence |
| --- | --- | --- |
| Cause is the blank credential under Moonraker 0.11.0 | **built and run** | `Unauthorized` present at 10:15, absent at 10:45 after the change, same machine, same log |
| Both files compile | **built and run** | `~/.KlipperScreen-env/bin/python -m py_compile` on both files printed `SYNTAX_OK` on the Pad |
| The commit is what is running | **built and run** | `git -C ~/KlipperScreen log --oneline -1` returned `059aecf (HEAD -> master, origin/master)` |
| Moonraker accepts the fork | **built and run** | update status: `is_valid True`, `is_dirty False`, `v0.4.3-16` to `v0.4.3-17` |
| Live temperatures restored | **built and run** | user confirmed by eye on the Pad |
| File browser lists gcodes | **built and run** | user confirmed by eye on the Pad |
| The `"False"` api_key normalisation works | **written, unverified** | no config on hand uses it |
| The `x-api-key` header path still authenticates | **written, unverified** | no api_key configured on this machine |
| `force_logins` setups work | **written, unverified** | not configured on this machine |

The last three are declared as untested in PR #51's body.

### Licence check

| Repo | Licence |
| --- | --- |
| `KlipperScreen/KlipperScreen` | AGPL-3.0 |
| `Guilouz/KlipperScreen-Flsun-Speeder-Pad` | AGPL-3.0 |
| the user's fork | AGPL-3.0, `LICENSE` intact at 662 lines |

Porting AGPL code into an AGPL project is permitted. `LICENSE` unchanged, source
public, all three upstream SHAs named in the commit message and linked in the PR,
no added restrictions. No violation.

## End state

**Working right now:** the Speeder Pad runs `059aecf` from the user's fork.
Live temps and the file browser both work. Moonraker reports the repo valid and
clean.

**PR #51** is open against `Guilouz/KlipperScreen-Flsun-Speeder-Pad`,
`VLoorenDeJong:master` into `Guilouz:master`, 1 commit, 2 files, +18 -11,
mergeable state `clean`, body opens with `Fixes #50`.

**Fragile, and the thing most likely to bite later:** the Pad now pulls
KlipperScreen from the user's personal fork, not Guilouz's. It will keep doing
so until somebody changes `moonraker.conf` back by hand. Nothing on the machine
or in any repo records this. If the user forgets, they silently stop receiving
Guilouz's updates.

Secondary: "Allow edits by maintainers" is on for PR #51 and the PR head is the
fork's `master`, which the Pad tracks. A push by Guilouz to that branch would
reach the Pad on its next update.

**Not broken, nothing needs rescuing.**

## Reproduction

From a Speeder Pad showing frozen stats and an empty file browser:

1. Confirm the cause:
   `tail -n 60 ~/printer_data/logs/KlipperScreen.log` and look for the
   `-32602 Unauthorized` callback after `server.files.list`
2. Confirm the Moonraker version is 0.11.0 or later, in the same log's
   `Moonraker info` line
3. Fork `Guilouz/KlipperScreen-Flsun-Speeder-Pad` on GitHub (browser: no `gh`
   CLI on the workstation)
4. Clone the fork. Note the files are CRLF
5. Apply the four edits listed under Changes, to
   `ks_includes/KlippyWebsocket.py` and `screen.py`
6. Review with `git --no-pager diff`, expect 4 hunks, 2 files, +18 -11
7. Commit as one commit, push to the fork's `master`
8. On the Pad, `git -C ~/KlipperScreen status --porcelain` must print nothing.
   If it prints anything, run `git -C ~/KlipperScreen checkout -- .` first
9. Repoint the git remote at the fork with `git remote set-url`
10. Repoint the `origin:` line under `[update_manager KlipperScreen]` in
    `~/printer_data/config/moonraker.conf` at the same fork
11. `sudo systemctl restart moonraker`
12. Confirm `is_valid True` and `is_dirty False` via the
    `/machine/update/status?refresh=true` endpoint on port 7125
13. Mainsail, then Machine, Update Manager, KlipperScreen, Update
14. Compile-check both edited files with the KlipperScreen virtualenv's python
15. `sudo systemctl restart KlipperScreen`, wait 20s, check the log has no new
    `Unauthorized`, and look at the physical screen

## Open

1. **Post the issue #50 comment.** Written and ready at
   `.claude/docs/pr-texts/2-issue-50-comment.md`, `#51` already filled in.
   Never posted.
2. **Commit and push the untracked work in this repo.** `CLAUDE.md`,
   `.gitmodules`, the `.claude/guiderails` submodule pointer, `.claude/docs/`,
   and this note are all uncommitted on `main`. The user was not asked about
   pushing their own files.
3. **Watch PR #51.** Guilouz has not commented on issue #50 in over three
   weeks and his repo has had no commit since 2024-08-29. Expect it to sit.
4. **When #51 merges, revert the Pad to Guilouz.** Steps are in section C of
   `.claude/docs/pr-texts/2-issue-50-comment.md`. Until then the Pad is on the
   personal fork and receives no upstream updates.
5. **Untested paths, if anyone reports back on #50 or #51:**
   `moonraker_api_key` actually set, `force_logins`, and the `"False"`
   normalisation. None could be exercised here.
