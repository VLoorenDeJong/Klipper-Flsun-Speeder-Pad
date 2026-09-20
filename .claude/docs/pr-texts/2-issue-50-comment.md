Fix is up as #51. No file editing needed, and your repo stays clean.

## What is wrong, in plain terms

Moonraker 0.11.0 got stricter about checking who is connecting. KlipperScreen
hands it a **blank** ID badge. The old Moonraker shrugged; the new one treats
a blank badge as a failed attempt and withdraws the trust it had already given
([changelog](https://moonraker.readthedocs.io/en/latest/changelog/)).

So the screen connects, and then everything it asks for is refused:

```
files.py:_callback() - {'code': -32602, 'message': 'Unauthorized'}
```

Frozen temperatures and an empty file browser are **one** problem, not two.
Both go through that same refused connection.

## About @h3rm's workaround

It finds exactly the right two lines and it works. The one catch: editing the
files by hand marks the repo **DIRTY**, and then the Update Manager refuses to
install anything ever again.

#51 makes the same change as a proper commit instead, so updates keep working.

## Small print

- Ported with Claude Code, then reviewed and tested by me on my own printer
- The three upstream commits it comes from are linked in #51, so anyone can
  check it line by line
- Tested on one FLSUN V400 Speeder Pad with **no** api_key set
- **Not tested:** setups with `moonraker_api_key` filled in, or `force_logins`.
  If that is you, please try it and say whether it works

---

## Instructions

Pick the section that matches you. Each one is click-to-expand.

<details>
<summary><b>A. Did you already hand-edit the files? Start here</b></summary>

<br>

If you applied the workaround by editing `KlippyWebsocket.py`, those edits
have to go first, or nothing will install. Do **not** use the Recover button.

**A1.** See what you changed:

```shell
git -C ~/KlipperScreen status --porcelain
```

> Expect a line like `M ks_includes/KlippyWebsocket.py`.
> If nothing prints, you have no edits: skip to section B.

**A2.** Undo them:

```shell
git -C ~/KlipperScreen checkout -- .
```

> Expect no output.

**A3.** Confirm it worked:

```shell
git -C ~/KlipperScreen status --porcelain
```

> Expect nothing at all. If something still prints, stop and post it here.

**A4.** Restart the screen:

```shell
sudo systemctl restart KlipperScreen
```

> Expect no output. The screen will be broken again, which is correct.
> Section B fixes it.

</details>

<details>
<summary><b>B. Install the fix now, before it is merged</b> (about 5 minutes)</summary>

<br>

**B1.** Check nothing is edited:

```shell
git -C ~/KlipperScreen status --porcelain
```

> Expect nothing. If anything prints, do section A first.

**B2.** Point KlipperScreen at the fork:

```shell
git -C ~/KlipperScreen remote set-url origin https://github.com/VLoorenDeJong/KlipperScreen-Flsun-Speeder-Pad.git
```

> Expect no output.

**B3.** Check that took:

```shell
git -C ~/KlipperScreen remote get-url origin
```

> Expect: `https://github.com/VLoorenDeJong/KlipperScreen-Flsun-Speeder-Pad.git`

**B4.** Tell Moonraker the same thing:

```shell
sed -i 's|Guilouz/KlipperScreen-Flsun-Speeder-Pad|VLoorenDeJong/KlipperScreen-Flsun-Speeder-Pad|' ~/printer_data/config/moonraker.conf
```

> Expect no output.

**B5.** Check that took:

```shell
grep -n "KlipperScreen-Flsun-Speeder-Pad" ~/printer_data/config/moonraker.conf
```

> Expect one line, an `origin:` with `VLoorenDeJong` in it.

**B6.** Restart Moonraker:

```shell
sudo systemctl restart moonraker
```

> Expect no output.

**B7.** Confirm Moonraker is happy and sees the update:

```shell
curl -s "http://127.0.0.1:7125/machine/update/status?refresh=true" | python3 -c "import sys,json; d=json.load(sys.stdin)['result']['version_info']['KlipperScreen']; print({k:d.get(k) for k in ('is_valid','is_dirty','version','remote_version')})"
```

> Expect:
> `{'is_valid': True, 'is_dirty': False, 'version': 'v0.4.3-16', 'remote_version': 'v0.4.3-17'}`
>
> If `is_dirty` is `True`, go back to section A.

**B8.** In Mainsail: **Machine** -> **Update Manager** -> **KlipperScreen** ->
**Update**.

> It pulls the fix, reinstalls what it needs, and restarts the screen itself.
> The version then reads `v0.4.3-17`.

**B9.** Final check, with the screen running:

```shell
grep Unauthorized ~/printer_data/logs/KlipperScreen.log
```

> Expect no new lines after the restart. Old ones from before the fix are
> fine to see.

Then look at the Pad: temperatures should move, and the file browser should
list your gcodes.

</details>

<details>
<summary><b>C. Switching back to Guilouz once this is merged</b></summary>

<br>

Do not do this before #51 is merged, or step C3 puts the bug back.

**C1.** Point KlipperScreen back:

```shell
git -C ~/KlipperScreen remote set-url origin https://github.com/Guilouz/KlipperScreen-Flsun-Speeder-Pad.git
```

> Expect no output.

**C2.** Tell Moonraker the same:

```shell
sed -i 's|VLoorenDeJong/KlipperScreen-Flsun-Speeder-Pad|Guilouz/KlipperScreen-Flsun-Speeder-Pad|' ~/printer_data/config/moonraker.conf
```

> Expect no output.

**C3.** Match Guilouz's history again. This throws away any local changes in
`~/KlipperScreen`:

```shell
git -C ~/KlipperScreen fetch origin && git -C ~/KlipperScreen reset --hard origin/master
```

> Expect some fetch output, then `HEAD is now at ...` naming Guilouz's latest
> commit.

**C4.** Restart Moonraker:

```shell
sudo systemctl restart moonraker
```

> Expect no output.

**C5.** Confirm it is back and healthy:

```shell
curl -s "http://127.0.0.1:7125/machine/update/status?refresh=true" | python3 -c "import sys,json; d=json.load(sys.stdin)['result']['version_info']['KlipperScreen']; print({k:d.get(k) for k in ('is_valid','is_dirty','remote_url')})"
```

> Expect `is_valid` `True`, `is_dirty` `False`, and a `remote_url` with
> `Guilouz` in it.

</details>
