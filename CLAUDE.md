# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`buds3` is a single self-contained bash script (no build system, package manager,
or test suite) that controls a Bluetooth headset on Linux via BlueZ
(`bluetoothctl`) and PipeWire/PulseAudio (`pactl`). It was written for the
OnePlus Buds 3 but works with any headset exposing the same profile names.

## Development

There is no build, lint, or test tooling in this repo. Useful checks:

```bash
shellcheck buds3        # static analysis (if installed)
bash -n buds3           # syntax check
./buds3 status          # exercise against a real paired device
```

Manual testing requires an actual paired Bluetooth headset — behavior can't be
meaningfully unit tested since it shells out to `bluetoothctl`/`pactl` and
depends on live device/audio-card state.

## Architecture

The whole script (`buds3`) is organized in layers, top to bottom:

- **Config constants**: `DEVICE_NAME`, `A2DP_PROFILE`, `HFP_PROFILE` — the
  only values a user typically needs to change to target a different headset.
- **Discovery helpers**: `resolve_device` looks up the MAC address from
  `bluetoothctl devices Paired` and derives the PipeWire card name
  (`bluez_card.<MAC_with_underscores>`). Nearly every other function depends
  on `MAC`/`CARD` being set first via `resolve_device`.
- **State query helpers**: `is_bt_connected`, `active_profile`, `card_exists`,
  `bt_sink_name`/`bt_source_name`, `sink_exists`/`source_exists`,
  `default_sink`/`default_source` — all thin wrappers around `bluetoothctl`/
  `pactl` output parsing (mostly `awk`).
- **Default sink/source persistence**: `save_prev_defaults` snapshots the
  current non-Bluetooth default sink/source to
  `$XDG_STATE_HOME/buds3/prev_defaults` (only once — it won't overwrite a
  saved value with a bluez default when switching modes while connected).
  `apply_audio_defaults` points defaults at the bluez sink/source once
  connected (falling back to the saved non-bt source when the mode has no
  mic, e.g. A2DP). `restore_audio_defaults` puts the saved defaults back on
  disconnect and clears the state file.
- **Actions**: `do_connect <profile> <label>`, `do_disconnect`, and
  `print_status` compose the helpers above into the full command flow, each
  printing a "current status" / progress / "final status" sequence.
- **Dispatch**: a `case` on `$1` at the bottom maps CLI subcommands
  (`a2dp`/`music`, `hfp`/`call`, `disconnect`/`off`, `status`) to the actions.

Key invariants:
- Mode switching (`pactl set-card-profile`) is used instead of a full
  reconnect when the device is already connected — reconnecting is only done
  when Bluetooth itself is disconnected.
- `wait_for <timeout_s> <check_cmd...>` polls every 0.5s and is used
  throughout to tolerate async delays (BlueZ connect, PipeWire card
  appearing, bluez sink appearing after profile switch).
