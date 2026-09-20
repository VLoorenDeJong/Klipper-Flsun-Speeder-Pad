Fixes #50.

## Problem

Moonraker v0.11.0 ([changelog](https://moonraker.readthedocs.io/en/latest/changelog/)):

> Failed JWT and API Key authentication attempts will now revoke "trusted
> client" authorization if present.

With no api_key configured, KlipperScreen sends a **blank** credential twice:

- `?token=` on the websocket URL
- `"api_key": ""` in `server.connection.identify`

Both now count as failed attempts and revoke trusted-client status, so the
websocket opens and every call over it is refused:

```
files.py:_callback() - {'code': -32602, 'message': 'Unauthorized'}
```

One fault, two symptoms:

- frozen temperatures: `printer.objects.subscribe` never delivers
- empty file browser: `server.files.list` never returns

## Changes

- Drop `?token=` from the websocket URL. `x-api-key` already carries the key
- Send `api_key` in `identify` only when one is set, instead of `""`
- Treat a literal `"False"` api_key as blank
- Add `User-Agent: KlipperScreen`

The first two are the lines @h3rm found in #50, applied as a commit so the
repo does not go DIRTY and updates keep working.

## Source

Three commits from [KlipperScreen#1760](https://github.com/KlipperScreen/KlipperScreen/issues/1760), squashed into one:

- [`f081f586`](https://github.com/KlipperScreen/KlipperScreen/commit/f081f586e0aa6b92177c8982359089065e7dad90) header-only API key auth
- [`27555e80`](https://github.com/KlipperScreen/KlipperScreen/commit/27555e80e842fc34358a7ad65c90e081dcb5e3e8) accept False for a blank api_key
- [`0ccec8be`](https://github.com/KlipperScreen/KlipperScreen/commit/0ccec8bec1c3f320abf7e87b0c3307220a11bbba) send the agent

Not cherry-pickable: upstream renamed the API layer, so two of them touch
`MoonrakerApi.py` and `KlippyUDS.py`, absent here.

**Ported with Claude Code**, reviewed and tested by me on the hardware below.
The commit carries a `Co-Authored-By` trailer.

## Testing

Tested:

- FLSUN V400 Speeder Pad, no api_key set
- Moonraker `v0.11.0-1-g1cfb0c4`, Klipper `v0.13.0-770-gce7002be`
- installed via the update manager, `v0.4.3-16` to `v0.4.3-17`
- `Unauthorized` gone from the log
- live temperatures and file browser confirmed on the screen

Not tested, no hardware for them:

- `moonraker_api_key` actually set
- `force_logins`
- the `"False"` case

Happy to reshape: three commits, rebased, or the first two points only.
