# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`bt` is a single self-contained bash script (no build system, package manager,
or test suite) that lists and controls Bluetooth audio devices on Linux via
BlueZ (`bluetoothctl`) and PipeWire/PulseAudio (`pactl`). Any paired device is
selected by name on the command line; it was originally written for the OnePlus
Buds 3 but is device-agnostic and auto-selects the best codec profile each
device exposes.

## Development

There is no build, lint, or test tooling in this repo. Useful checks:

```bash
shellcheck bt           # static analysis (if installed)
bash -n bt              # syntax check
./bt                    # list devices (safe; needs only paired devices)
./bt <name> status      # exercise against a real paired device
```

Manual testing requires an actual paired Bluetooth headset — behavior can't be
meaningfully unit tested since it shells out to `bluetoothctl`/`pactl` and
depends on live device/audio-card state.

## Architecture

The whole script (`bt`) is organized in layers, top to bottom:

- **Discovery helpers**: `list_devices` emits `<MAC>\t<Name>` for every known
  device (from `bluetoothctl devices`). `resolve_device <query>` matches the
  query case-insensitively as a substring of the name (or exact MAC), erroring
  on no match and on ambiguity (unless an exact full-name match disambiguates);
  it sets the globals `MAC`, `DEVICE_NAME`, `CARD`
  (`bluez_card.<MAC_with_underscores>`), and the per-device `STATE_FILE`.
  Nearly every other function depends on these being set first.
- **Profile selection**: there are no hardcoded profile constants.
  `available_profiles` parses the card's `Profiles:` block for currently
  `available: yes` profile names; `pick_profile <a2dp|hfp>` chooses the best
  one from a per-kind preference list (e.g. a2dp: sbc_xq > ldac > aptx_hd >
  aptx > aac > plain), falling back to any profile matching `^a2dp-sink` /
  `^headset`. This is what makes the script device-agnostic.
- **State query helpers**: `is_bt_connected`, `active_profile`, `card_exists`,
  `bt_sink_name`/`bt_source_name`, `sink_exists`/`source_exists`,
  `default_sink`/`default_source` — all thin wrappers around `bluetoothctl`/
  `pactl` output parsing (mostly `awk`).
- **Default sink/source persistence**: `save_prev_defaults` snapshots the
  current non-Bluetooth default sink/source to
  `$XDG_STATE_HOME/bt/prev_defaults.<MAC>` (per device, only once — it won't
  overwrite a saved value with a bluez default when switching modes while
  connected). `apply_audio_defaults` points defaults at the bluez sink/source
  once connected (falling back to the saved non-bt source when the mode has no
  mic, e.g. A2DP). `restore_audio_defaults` puts the saved defaults back on
  disconnect and clears the state file.
- **Actions**: `do_list` prints all known devices with a connection marker;
  `do_connect <a2dp|hfp> <label>` (resolves the concrete profile via
  `pick_profile`), `do_disconnect`, and `print_status` compose the helpers
  above into the full command flow, each printing a "current status" /
  progress / "final status" sequence. `do_clean` (`bt clean`/`prune`)
  `bluetoothctl remove`s every known device that isn't in `bluetoothctl
  devices Paired` — the scan-cache junk (nearby BLE advertisers, stale re-pair
  addresses) that accumulates in the list — and warns if the adapter is still
  `Discovering: yes`, since live advertisers get re-cached the instant they're
  removed (blueman-applet / an open Bluetooth settings panel is the usual
  culprit holding discovery open).
- **Dispatch**: a `case` on `$1` at the bottom routes the no-arg/`list` form to
  `do_list`, `clean`/`prune` to `do_clean`, and otherwise treats `$1` as the
  device query (via `resolve_device`) and `$2` as the action (`a2dp`/`music`,
  `hfp`/`call`, `disconnect`/`off`, `status`; defaulting to `status`).

Key invariants:
- Mode switching (`pactl set-card-profile`) is used instead of a full
  reconnect when the device is already connected — reconnecting is only done
  when Bluetooth itself is disconnected.
- `wait_for <timeout_s> <check_cmd...>` polls every 0.5s and is used
  throughout to tolerate async delays (BlueZ connect, PipeWire card
  appearing, bluez sink appearing after profile switch).

## Troubleshooting

- **`connect` keeps timing out though the device is paired/trusted and
  powered on** (often while the device shows a pairing/breathing light): the
  stored BlueZ bond is stale — the device is advertising for a *fresh* pair,
  not a reconnect, so `bluetoothctl connect` never completes. `bt` can't fix
  this itself; drop the old bond and re-pair while the device is discoverable:

  ```bash
  bluetoothctl remove <MAC>
  bluetoothctl --timeout 15 scan on    # rediscover it
  bluetoothctl pair <MAC>
  bluetoothctl trust <MAC>
  bluetoothctl connect <MAC>
  bt <dev> music                       # then route audio via bt
  ```

  Note: pairing a new device can make the adapter drop an already-connected
  one; reconnect it afterward with `bt <dev> music`.

- **The device list keeps filling with junk (random-MAC BLE advertisers,
  trackers, a *duplicate* of a real headset under a rotating address) and
  `bt clean` removals don't stick**: something is holding the adapter in
  active discovery, so live advertisers get re-cached the instant they're
  removed. Confirm with `bluetoothctl show | grep Discovering` (`yes`). The
  usual suspects are `blueman-applet` or an open Bluetooth settings panel —
  but a **browser doing Web Bluetooth** (`navigator.bluetooth`, e.g. a Brave/
  Chrome tab scanning for BLE devices) is a common, non-obvious cause and
  overrides everything: killing blueman, `bluetoothctl power off/on`, and even
  `systemctl restart bluetooth` won't stop it, because the browser re-issues
  `StartDiscovery` the moment the adapter reappears. `bluetoothctl scan off`
  fails with `org.bluez.Error.Failed` since the scan is owned by another
  client, not `bt`. Find the owning process by tracing the D-Bus call:

  ```bash
  busctl monitor org.bluez          # watch for Member=StartDiscovery; note Sender=:1.N
  busctl status :1.N                # map that unique name -> PID/Comm
  ```

  (Provoke a fresh `StartDiscovery` by cycling the adapter — `bluetoothctl
  power off; power on` — while monitoring.) For a browser, close the scanning
  tab or revoke its Bluetooth permission (`brave://bluetooth-internals` lists
  the active scan's origin); for blueman, quit the applet. Once `Discovering:
  no`, `bt clean` sticks. The clutter is purely cosmetic — it never affects
  connecting or audio — so living with it and running `bt clean` on demand is
  a valid option too.
