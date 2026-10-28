# bt

A small bash script to list and control Bluetooth audio devices on Linux with
BlueZ + PipeWire/PulseAudio. Pick any paired device by name, connect it in one
of two audio modes, disconnect, or report status — switching the system default
output/input (sink/source) to match, and restoring your previous defaults on
disconnect.

It was originally written for the OnePlus Buds 3 but works with any
headset/speaker that exposes standard BlueZ/PipeWire profiles; the right codec
profile is chosen automatically per device.

Modes:

1. **A2DP Sink** — high-quality stereo, no mic (best available codec:
   SBC-XQ / LDAC / aptX / AAC / SBC)
2. **HSP/HFP** — mono, with mic, for calls (best available codec: mSBC / CVSD)
3. **Disconnected**

## Requirements

- Linux with [BlueZ](http://www.bluez.org/) (`bluetoothctl`)
- PipeWire with `pipewire-pulse` (provides `pactl`), or PulseAudio
- The device already paired and trusted with the system (via `bluetoothctl`
  or your desktop's Bluetooth settings)

## Install

```bash
git clone git@github.com:jemshid/buds3.git
cd buds3
mkdir -p ~/.local/bin
cp bt ~/.local/bin/
chmod +x ~/.local/bin/bt
```

Make sure `~/.local/bin` is on your `PATH` (add to `~/.bashrc` / `~/.zshrc`
if not):

```bash
export PATH="$HOME/.local/bin:$PATH"
```

No per-device configuration is needed — the device is selected by name on the
command line and the best codec profile it exposes is chosen automatically.

## Usage

```bash
bt                       # list known devices and their connection state
bt <name> a2dp           # or: bt <name> music
bt <name> hfp             # or: bt <name> call
bt <name> disconnect       # or: bt <name> off
bt <name> status          # default action if omitted
```

`<name>` is a case-insensitive substring of the device name (or its MAC), so
`bt beats a2dp`, `bt buds hfp`, or `bt jbl off` all work. If the substring
matches more than one device, `bt` lists the matches so you can be more
specific.

Each control command prints the current status, what it's doing, and the
resulting status — including Bluetooth connection state, active audio profile,
and default output/input devices — so you can see exactly what changed.

### Example

```
$ bt
Known Bluetooth devices:
  ● OnePlus Buds 3               40:72:18:22:72:8B
  ○ BeatsPill+                   78:4F:43:CF:74:42
  ○ JBL TUNE215TWS               14:2C:78:03:40:CD

Control one with: bt <name> a2dp|hfp|off|status

$ bt buds hfp
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

- Devices are discovered from `bluetoothctl devices` and matched by name/MAC
  substring.
- Connection is handled via `bluetoothctl connect`/`disconnect`.
- The audio mode's codec profile is picked automatically from the profiles
  the card actually exposes (`pactl list cards`), preferring higher-quality
  codecs.
- Mode switching is done by setting the PipeWire/PulseAudio card profile
  (`pactl set-card-profile`) — if already connected, this switches modes
  without a full Bluetooth reconnect.
- Default sink/source are set with `pactl set-default-sink` /
  `set-default-source`. Before first connecting, the script saves your
  current (non-Bluetooth) defaults to
  `~/.local/state/bt/prev_defaults.<MAC>` (one file per device), and restores
  them on disconnect.

## License

MIT
