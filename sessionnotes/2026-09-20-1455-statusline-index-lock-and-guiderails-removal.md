# Session: 2026-09-20 14:55: stale index.lock cause found, guiderails removed

| | |
| --- | --- |
| Started / ended | 2026-09-20, ~14:35 to ~14:55 (Europe/Amsterdam) |
| Repo / branch | `VLoorenDeJong/Klipper-Flsun-Speeder-Pad` on `main`, and `VLoorenDeJong/Guiderails` on `main` |
| Machines touched | Windows 11 workstation only. The Speeder Pad was NOT touched this session |
| Commits | `754248c` (this repo), `17ec36b` (Guiderails) |
| Pushed | Yes, both, verified by comparing HEAD to origin |

## Goal

Follow up the previous session
(`2026-09-20-1500-klipperscreen-moonraker-011-auth.md`): check its open loops
against the live internet state, store the findings, then remove the guiderails
submodule from this repo cleanly.

## What happened

### 1. Verified the previous session's open loops against GitHub

All checks read through the public GitHub API. There is no `gh` CLI on this
workstation.

| Open loop from last session | Live state on 2026-09-20 | Verdict |
| --- | --- | --- |
| Post the issue #50 comment | Comment 4 by `VLoorenDeJong`, posted 2026-09-20T09:31:28Z | **Done.** The user had already posted it |
| PR #51 state | Open, `mergeable_state: clean`, 0 comments, 0 reviews, untouched since 09:26:16Z | Still open, no maintainer response |
| Fork `master` HEAD | `059aecf8288d37b219ff26afe2059954f56f0fff` | Matches the previous note exactly |

Nothing in the previous session's record turned out to be wrong.

### 2. Root cause found for the recurring `.git/index.lock` failures

The user reported this had been happening for about three days across repos. It
blocked the first `git add` of this session.

Evidence gathered:

| Observation | Conclusion |
| --- | --- |
| Lock in this repo, 0 bytes, dated 09:54 | A process created the file, then died before writing |
| Second lock in `E:/GitHubRepos/LinuxSetups`, 0 bytes, dated 14:44 | Appeared minutes earlier, in a repo this session never touched |
| `tasklist` showed no `git.exe`, checked twice 70s apart | Both locks were dead, not in-progress |
| `~/.claude/statusline.js:70` ran `git status --porcelain` with `timeout: 2000` | The lock-taker |
| `statusline.js` installed 2026-09-14 20:25 by `setup_guiderails.sh` | Matches the recent onset |

Mechanism:

```
statusline renders -> git status --porcelain -> git creates .git/index.lock
                                             -> 2000ms timeout kills it
                                             -> 0-byte index.lock orphaned
```

A plain `git status` takes `index.lock` to refresh the index stat cache. Killed
mid-refresh, the lock outlives the process. Every later git command in that repo
then fails with `Unable to create '.git/index.lock': File exists`.

Defender real-time protection is ON and its exclusion list cannot be read
without admin. That is the plausible reason `git status` exceeds 2 seconds on a
465 GB NTFS volume, but it was NOT proven. The kill comes from the statusline's
own timeout, and that part is proven.

### 3. The fix, and a wrong first attempt

The first attempt put the flag in the wrong place:

```
git status --no-optional-locks --porcelain    ->  error: unknown option
```

`--no-optional-locks` is a git-level option, not a `git status` option, so it
belongs before the subcommand:

```
git --no-optional-locks status --porcelain
```

This was caught because the flag was tested against real git before being
trusted, not after.

## Causes

`--no-optional-locks` tells git to skip the optional index refresh write. A
read-only status then cannot create `index.lock` at all, so killing it mid-run
cannot orphan anything.

The 2000ms timeout was deliberately left in place. It is no longer harmful, and
raising it would have been a second change with no evidence that it was needed.

## Ruled out

| Hypothesis | How it was eliminated |
| --- | --- |
| VS Code's Git extension | The leading suspect before the statusline was found. The statusline explains both the recent onset and the 0-byte signature; VS Code has been installed far longer |
| A crashed terminal from an earlier session | Explains the 09:54 lock but not the 14:44 one, in a repo nobody had open |
| Antivirus alone | Defender makes `git status` slow, but the kill comes from the timeout. Defender alone would not produce a 0-byte lock |
| `.claude/docs` needed archiving into the Guiderails submodule | `setup_guiderails.sh:270` leaves an existing `project-context.md` untouched, and `clean_guiderails.sh:146` keeps one with real content. Committing the docs to this repo already survives both removal and reinstall |

## Changes

### Repo `VLoorenDeJong/Guiderails`, branch `main`

Commit `17ec36b36b6c16dbb96b67a4ca4565b4058610f5`,
"fix: stop the statusline orphaning .git/index.lock". 1 file, 1 line.

- `tooling/statusline.js:70`: `git status --porcelain` became
  `git --no-optional-locks status --porcelain`

### Repo `VLoorenDeJong/Klipper-Flsun-Speeder-Pad`, branch `main`

Commit `754248cead22a21add246e9a1ef57033c07079b9`,
"docs: keep the Moonraker v0.11.0 session record". 4 files, +817.

- `sessionnotes/2026-09-20-1500-klipperscreen-moonraker-011-auth.md`
- `.claude/docs/klipperscreen-moonraker-0.11-findings.md`
- `.claude/docs/pr-texts/1-pr-description.md`
- `.claude/docs/pr-texts/2-issue-50-comment.md`

### Changes on the workstation, not in any repo

- `C:/Users/Victor/.claude/statusline.js:70`: the same one-line fix applied to
  the installed copy. This file is per-user and is not tracked by any repo. If
  it is ever reinstalled from an older Guiderails checkout, the bug returns
- Deleted the stale
  `E:/GitHubRepos/Klipper-Flsun-Speeder-Pad/.git/index.lock`. The `LinuxSetups`
  one had already disappeared by the time it was removed, by something outside
  this session

### Guiderails removed from this repo

`bash .claude/guiderails/clean_guiderails.sh --yes` removed, in its own words:

1. `CLAUDE.md` (nothing left but the stub setup created)
2. `.claude/docs/project-context.md` (wizard marker still pending, so an empty stub)
3. The `.claude/guiderails` submodule (git rm + deinit)
4. `.git/modules/.claude/guiderails`

Three things the script did NOT clean, removed by hand afterwards:

| Leftover | Why the script missed it |
| --- | --- |
| `.gitmodules`, still holding the full submodule stanza | The submodule was never committed, only staged, and a `git reset` earlier in this session unstaged it. The script's `git rm` therefore had nothing in the index to act on |
| `.git/config` keys `submodule..claude/guiderails.url` and `.active` | `deinit` did not strip them under the same condition |
| `.git/config` key `submodule.active` | Same |

That `git reset` was mine, run to stage only the four files worth keeping. It is
the reason the uninstall came out partial. Anyone re-running
`clean_guiderails.sh` after unstaging a never-committed submodule should expect
the same, and should check `git config --get-regexp submodule` afterwards.

## Verification

No evidence ledger in this repo, so this section is written by hand.

| Claim | Status | Evidence |
| --- | --- | --- |
| The statusline was the lock-taker | **built and run** | `git --no-optional-locks status --porcelain` returned normally in this repo while the stale `index.lock` was still present. The unflagged form fails there |
| The corrected flag is valid for git 2.55.0.windows.4 | **built and run** | Ran it, exit 0. The first ordering returned `error: unknown option` |
| `statusline.js` still parses and renders | **built and run** | `node --check` printed SYNTAX_OK; piping a workspace JSON in printed `Klipper-Flsun-Speeder-Pad / main (6 changes)` |
| The Guiderails fix is on the remote | **built and run** | `git rev-parse HEAD origin/main` returned the same SHA twice |
| The session record is on the remote | **built and run** | Same check, `754248c...` twice |
| This repo has no submodule trace left | **built and run** | `git config --get-regexp submodule` prints NONE; `git status --porcelain` prints only the pre-existing `?? .vscode/` |
| Defender is why `git status` exceeded 2s | **unverified** | The exclusion list needs admin. The timeout kill is proven; the slowness cause is not |
| The fix stops new locks appearing over time | **written, unverified** | Needs days of normal use. Nothing here can prove a negative in 20 minutes |

## End state

**Working right now:** the Speeder Pad still runs `059aecf` from the user's
fork, untouched this session. This repo is back to a plain config repo with no
guiderails wiring and no `CLAUDE.md`, plus one new commit holding the session
record. Guiderails carries the statusline fix.

**Fragile, unchanged from last session:** the Pad pulls KlipperScreen from the
user's personal fork, not Guilouz's.
`~/printer_data/config/moonraker.conf` line 60 records this and nothing else
does. It must be reverted by hand when PR #51 merges.

**New fragile thing:** `C:/Users/Victor/.claude/statusline.js` is a per-user
file holding a fix that also lives in Guiderails `17ec36b`. Any other machine
still running the old statusline keeps orphaning locks.

**Not broken, nothing needs rescuing.**

## Reproduction

To confirm the index.lock diagnosis on another machine:

1. `ls -l .git/index.lock` in an affected repo. A 0-byte file is the signature
2. `tasklist //FI "IMAGENAME eq git.exe"`. No results means the lock is dead
3. `grep -n "git status" ~/.claude/statusline.js`. An unflagged
   `git status --porcelain` carrying a `timeout` is the cause
4. Fix it by inserting `--no-optional-locks` before `status`, not after
5. `node --check ~/.claude/statusline.js`
6. Prove it: with a stale `index.lock` still in place, run
   `git --no-optional-locks status --porcelain`. It returns normally where the
   unflagged form fails

## Open

1. **Watch PR #51.** Open, clean, zero maintainer activity. Guilouz's repo has
   had no commit since 2024-08-29. Expect it to sit
2. **When #51 merges, revert the Pad to Guilouz.** Steps are in section C of
   `.claude/docs/pr-texts/2-issue-50-comment.md`. Until then the Pad receives no
   upstream KlipperScreen updates
3. **Confirm no new 0-byte locks appear** over the next few days. That is the
   only real proof the statusline fix worked
4. **Propagate the statusline fix** to any other machine whose
   `~/.claude/statusline.js` predates Guiderails `17ec36b`
5. **Untested KlipperScreen paths**, if anyone reports on #50 or #51: a real
   `moonraker_api_key`, `force_logins`, and the `"False"` normalisation
