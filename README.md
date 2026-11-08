# Separate internal speaker and headphone jack on Realtek ALC221 (HP desktop) on Linux

Make the **internal speaker** and the **front headphone jack** act as two independent playback devices, so you can send different apps to each (the Linux equivalent of Realtek's "multi-streaming" on Windows). Includes the fix for the **headset microphone**, which the reconfig breaks.

Tested on one machine only:

| | |
|---|---|
| Distro | CachyOS (Arch), XFCE |
| Audio | PipeWire + WirePlumber |
| Codec | Realtek ALC221 (`Vendor Id: 0x10ec0221`) |
| Subsystem ID | `0x103c3397` (HP desktop, Core i5-3470) |
| Kernel sound card | `card0`, codec `hwC0D0` |

It may work on other HDA codecs that have two DACs, but that is **untested**. Check your own codec dump first (see "Diagnose first").

## The problem

- `pavucontrol` shows "Speakers" and "Headphones" as two ports of **one** sink. Selecting one mutes the other, and the volume is shared.
- With a plug in the headphone jack, the speaker is silent or cuts out after about a second.
- On Windows (open-source Realtek driver + SteelSeries Sonar) the two outputs could be used independently.

Things that did **not** fix it:

- Unmuting everything in `alsamixer`, and `Auto-Mute Mode = Disabled`
- `hdajackretask` boot override
- PipeWire/WirePlumber suspend settings
- Switching card profiles (Analog Duplex / Pro Audio) on their own
- `snd_hda_intel.model=...` on the kernel command line (this machine dropped into emergency mode; the exact cause was not confirmed)

## What actually works

Two things are needed, both applied **at runtime** through sysfs (no kernel parameters, no initramfs changes):

1. **`indep_hp = yes`** hint. Creates an `Independent HP` control and turns PCM device `00-02` ("ALC221 Alt Analog") into a second playback device.
2. **Tell the driver the rear Line Out pin (0x14) is not connected** (`user_pin_configs`). Without this, the driver treats Line Out as the primary output, and `hw:0,0` drives both the speaker DAC and the headphone DAC, so the headset hears everything.

Result on the test machine:

- `hw:0,0` plays on the **speaker**
- `hw:0,2` plays on the **headset**

Side effect: the **rear green Line Out jack stops working** (the pin is disabled). The front headphone jack is unaffected.

### Why it works (from the codec dump)

The codec has two DACs:

- DAC `0x02` is wired to the speaker pin `0x17` (via `0x0c`)
- DAC `0x03` is wired to the headphone pin `0x21` and the rear Line Out pin `0x14` (via `0x0d`)

With `indep_hp` the second PCM uses DAC `0x03`. Disabling pin `0x14` stops the first PCM from also feeding that path.

## Diagnose first

```bash
grep -E "Codec:|Subsystem Id" /proc/asound/card0/codec#0
cat /proc/asound/pcm
amixer -c 0 contents | grep -i indep
```

Look for two `Audio Output` converters (nodes like `0x02` and `0x03`), with the headphone pin and the speaker pin on different ones. If your pin numbers differ from `0x14`, `0x17` and `0x21`, adjust accordingly.

## Quick test (runtime only, a reboot undoes it)

PipeWire must not hold the card, or `reconfig` fails with "Device or resource busy". Close audio apps first.

```bash
systemctl --user stop pipewire.socket pipewire-pulse.socket pipewire pipewire-pulse wireplumber

echo 'indep_hp = yes'   | sudo tee /sys/class/sound/hwC0D0/hints
echo '0x14 0x411111f0'  | sudo tee /sys/class/sound/hwC0D0/user_pin_configs
echo 1                  | sudo tee /sys/class/sound/hwC0D0/reconfig

# reconfig rebuilds the mixer: controls reset and may come back muted
amixer -c 0 cset name='Independent HP' Enabled
amixer -c 0 sset 'Auto-Mute Mode' Disabled
amixer -c 0 sset Master 100% unmute
amixer -c 0 sset PCM 100% unmute
amixer -c 0 sset Speaker 100% unmute
amixer -c 0 sset Headphone unmute

# listen with the headset plugged in
speaker-test -D hw:0,0 -c 2 -t sine -l 1   # expect: speaker
speaker-test -D hw:0,2 -c 2 -t sine -l 1   # expect: headset

systemctl --user start pipewire.socket pipewire-pulse.socket wireplumber
```

Notes:

- `Independent HP` resets to Disabled after every `reconfig`, so enable it again afterwards.
- After the pin change, the `Auto-Mute Mode` control had different items (`Disabled` / `Enabled`).
- If `fish` is your shell, `VAR=value` and heredocs don't work; use `bash` for the scripts below.

## Microphone

The playback work above did not touch the mic, and `reconfig` rebuilds **all** mixer controls, so the capture controls come back muted or at 0. Three things went wrong on the test machine:

1. **Capture was off.** `Capture` and `Capture,1` (one per capture device) were `[off]` at 0%.
2. **The `Mic` and `Line` playback switches are not capture controls.** They send the input straight into the output ("hear yourself"). Unmuting them caused repeating loud hits and a tripled music signal through feedback. Keep them **muted**.
3. **Discord was listening to a Monitor.** In the Pro Audio profile, `pavucontrol`'s Input Devices list (and apps like Discord) shows "Monitor of Built-in Audio Pro" entries. Those are copies of what the speaker is playing, not microphones. Symptom: the mic test plays back your music and never picks up your voice.

Check that the codec sees the mic plug (front jack in this case):

```bash
amixer -c 0 cget iface=CARD,name='Mic Jack'    # expect values=on with the mic plugged in
```

Find the real capture device. Pro Audio on this machine exposed two:

```bash
pactl list short sources
# alsa_input.pci-0000_00_1b.0.pro-input-0
# alsa_input.pci-0000_00_1b.0.pro-input-2
```

Record from each while tapping the mic, and see which one plays your voice back (turn the volume down first):

```bash
timeout 5 pw-record --target alsa_input.pci-0000_00_1b.0.pro-input-0 /tmp/m0.wav; pw-play /tmp/m0.wav
timeout 5 pw-record --target alsa_input.pci-0000_00_1b.0.pro-input-2 /tmp/m2.wav; pw-play /tmp/m2.wav
```

Then set it as the default input (use whichever one worked) and pick "Default" in your app:

```bash
pactl set-default-source alsa_input.pci-0000_00_1b.0.pro-input-2
```

Never choose anything with "Monitor" in its name.

Gain: `Capture` at 100% is +30 dB and `Mic Boost` 2 adds 24 dB, which is too hot. The values in the script below (Capture 60%, Mic Boost 1) are a starting point. If your voice is too quiet, raise `Mic Boost` to 2 or `Capture` to 70%. If you hear distortion or hiss, lower them.

## Make it permanent (boot-time systemd service)

This runs before login, so PipeWire isn't holding the card yet. It does not touch kernel parameters, `/etc/modprobe.d`, or the initramfs.

`/usr/local/bin/hda-split.sh`:

```bash
#!/bin/bash
# Wait for the codec to appear (up to 30 s)
for i in $(seq 1 30); do
  [ -e /sys/class/sound/hwC0D0/reconfig ] && break
  sleep 1
done

echo 'indep_hp = yes' > /sys/class/sound/hwC0D0/hints
echo '0x14 0x411111f0' > /sys/class/sound/hwC0D0/user_pin_configs
echo 1 > /sys/class/sound/hwC0D0/reconfig
sleep 2

set_mixer() {
  # playback
  amixer -c 0 -q cset name='Independent HP' Enabled
  amixer -c 0 -q sset 'Auto-Mute Mode' Disabled
  amixer -c 0 -q sset 'Loopback Mixing' Disabled
  amixer -c 0 -q sset Master 100% unmute
  amixer -c 0 -q sset PCM 100% unmute
  amixer -c 0 -q sset Speaker 100% unmute
  amixer -c 0 -q sset Headphone unmute

  # mic: the "Mic" and "Line" playback switches only loop the input
  # back into the output (feedback), so keep them muted
  amixer -c 0 -q sset Mic mute
  amixer -c 0 -q sset Line mute

  # capture: two capture controls, one per input device
  amixer -c 0 -q sset 'Input Source' Mic
  amixer -c 0 -q sset 'Input Source',1 Mic
  amixer -c 0 -q sset Capture 60% cap
  amixer -c 0 -q sset Capture,1 60% cap
  amixer -c 0 -q sset 'Mic Boost' 1
}

set_mixer
sleep 2
set_mixer   # second pass in case something reset a control
exit 0
```

If a control doesn't exist on your board, `amixer` prints an error that `-q` hides and the script carries on.

`/etc/systemd/system/hda-split.service`:

```ini
[Unit]
Description=Split ALC221 speaker and headphone outputs
After=sound.target alsa-restore.service
Before=systemd-user-sessions.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/hda-split.sh
TimeoutStartSec=60

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
sudo chmod +x /usr/local/bin/hda-split.sh
sudo systemctl daemon-reload
sudo systemctl enable hda-split.service
```

Check the result without rebooting:

```bash
systemctl --user stop pipewire.socket pipewire-pulse.socket pipewire pipewire-pulse wireplumber
sudo systemctl restart hda-split.service
amixer -c 0 sget Capture | grep "Front Left:"      # expect [on], about 60%
amixer -c 0 sget 'Mic Boost' | grep "Front Left:"  # expect 1
amixer -c 0 sget Mic | grep "Front Left:"          # expect [off]
systemctl --user start pipewire.socket pipewire-pulse.socket wireplumber
```

**Important:** the first version of this script did not unmute `Master` / `PCM` after `reconfig`, and the machine booted with no sound. Always restore the mixer after it.

If a control is still wrong after a reboot, something else may be restoring an old state after the service runs. The likely suspect is `alsa-restore.service`. Check the ordering with `systemctl status hda-split.service alsa-restore.service --no-pager`.

### Undo

```bash
sudo systemctl disable --now hda-split.service
sudo rm /etc/systemd/system/hda-split.service /usr/local/bin/hda-split.sh
```

## Per-app routing

Select the **Pro Audio** profile (pavucontrol → Configuration → Built-in Audio) to expose both analog playback devices (shown as "Built-in Audio Pro" and "Built-in Audio Pro 2" in `qpwgraph`), then pick the output per app in the Playback tab. WirePlumber remembers the choice per app.

> **Not fully verified:** the sysfs fix, the boot service and the microphone were confirmed working on the test machine. Per-app routing across reboots was only partly tested. Pro Audio does not manage the mixer, and the pavucontrol volume slider may not control hardware volume, so use each app's own volume.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `reconfig`: "Device or resource busy" | PipeWire/WirePlumber or an app has the card open. Stop them (including the `.socket` units) and check `fuser -v /dev/snd/*` |
| `hw:0,2` gives "Device or resource busy" | `Independent HP` is still Disabled |
| No sound after boot | Mixer controls came back muted after `reconfig`. Unmute Master, PCM, Speaker, Headphone |
| Both devices play on both outputs | Pin `0x14` wasn't disabled, or you reconfigured without re-applying `user_pin_configs` |
| Mic test plays your music, not your voice | The app is on a "Monitor" source. Choose a real `alsa_input...` device |
| Loud repeating hits / tripled sound while the mic is on | `Mic` or `Line` playback switch is unmuted (feedback), or the capture gain is too high |
| Mic silent | `Capture` / `Capture,1` is off or at 0%, or the wrong `pro-input-*` device is selected |
| Strawberry/GStreamer "failed to configure audio sink" | The app holds a dead connection to a sink that was recreated. Restart the app |
| Rear Line Out jack dead | Expected, because pin `0x14` is disabled by this fix |

## Things to avoid

- Boot-level changes (kernel `model=` parameters, `/etc/modprobe.d` patch files such as `hda-jack-retask`, rebuilding the initramfs) put the test machine into emergency mode twice. Prefer runtime and boot-service changes. Keep a Btrfs/Snapper snapshot before experimenting.
- Virtual sinks and PipeWire loopbacks do not help. They sit above a single ALSA device with one active output.
- VGA carries no audio, so a monitor's headphone jack is not an option over VGA. HDMI or DisplayPort would be.

## Fallback

If your codec can't do this, a cheap USB audio adapter appears in PipeWire as a second, fully independent sound card.

## Contributing

If you try this on another board, please open an issue with: your codec (the header of `/proc/asound/card0/codec#0`), subsystem ID, the pin numbers you changed, and whether `hw:0,0` / `hw:0,2` split cleanly.

## Credits

Diagnosed and tested by [@Galio2007](https://github.com/Galio2007) on real hardware, with analysis and drafting assistance from Claude (Anthropic). The mechanisms used (`indep_hp` hint, `user_pin_configs`, `reconfig`) come from the Linux kernel's HD-audio sysfs interface.
