# buds3

A small bash script to control a Bluetooth headset (originally written for the
OnePlus Buds 3) on Linux with BlueZ + PipeWire/PulseAudio. It connects in two
audio modes, disconnects, and reports status — switching the system default
output/input (sink/source) to match, and restoring your previous defaults on
disconnect.

Modes:

1. **A2DP Sink** — codec SBC-XQ (high quality stereo, no mic)
2. **HSP/HFP** — codec mSBC (mono, with mic — for calls)
3. **Disconnected**

## Requirements

- Linux with [BlueZ](http://www.bluez.org/) (`bluetoothctl`)
- PipeWire with `pipewire-pulse` (provides `pactl`), or PulseAudio
- `pactl` must report profiles named `a2dp-sink-sbc_xq` and
  `headset-head-unit-msbc` for your device (most modern BlueZ/PipeWire
  Bluetooth stacks expose these — check with `pactl list cards`)
- Your headset already paired and trusted with the system (via
  `bluetoothctl` or your desktop's Bluetooth settings)

## Install

```bash
git clone git@github.com:jemshid/buds3.git
cd buds3
mkdir -p ~/.local/bin
cp buds3 ~/.local/bin/
chmod +x ~/.local/bin/buds3
```

Make sure `~/.local/bin` is on your `PATH` (add to `~/.bashrc` / `~/.zshrc`
if not):

```bash
export PATH="$HOME/.local/bin:$PATH"
```

By default the script targets a device named `OnePlus Buds 3` (must already
be paired). To use it with a different headset, edit the `DEVICE_NAME`
variable near the top of the script, and confirm the exact profile names
your device exposes (`pactl list cards`) match `A2DP_PROFILE` /
`HFP_PROFILE`, adjusting them if needed.

## Usage

```bash
buds3 a2dp         # or: buds3 music
buds3 hfp           # or: buds3 call
buds3 disconnect     # or: buds3 off
buds3 status
```

Each command prints the current status, what it's doing, and the resulting
status — including Bluetooth connection state, active audio profile, and
default output/input devices — so you can see exactly what changed.

### Example

```
$ buds3 hfp
==> Current status:
OnePlus Buds 3 (40:72:18:22:72:8B)
  Bluetooth: disconnected
  Default output: alsa_output...sink
  Default input:  alsa_input...source

==> Connecting to OnePlus Buds 3...
✓ Bluetooth connected
==> Waiting for audio card...
==> Switching to HSP/HFP (mSBC)...
✓ Now in HSP/HFP (mSBC) mode
==> Updating default output/input...
✓ Default output -> bluez_output.40_72_18_22_72_8B.1
✓ Default input -> bluez_input.40_72_18_22_72_8B.0

==> Final status:
OnePlus Buds 3 (40:72:18:22:72:8B)
✓ Bluetooth: connected
  Mode: HSP/HFP (mSBC)
  Default output: bluez_output.40_72_18_22_72_8B.1
  Default input:  bluez_input.40_72_18_22_72_8B.0
```

## How it works

- Connection is handled via `bluetoothctl connect`/`disconnect`.
- Mode switching is done by setting the PipeWire/PulseAudio card profile
  (`pactl set-card-profile`) — if already connected, this switches modes
  without a full Bluetooth reconnect.
- Default sink/source are set with `pactl set-default-sink` /
  `set-default-source`. Before first connecting, the script saves your
  current (non-Bluetooth) defaults to
  `~/.local/state/buds3/prev_defaults`, and restores them on disconnect.

## License

MIT
