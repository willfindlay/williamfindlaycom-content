---
title: "Fixing Claude Code Voice Mode on WSL2"
date: "2026-03-11T19:00:00Z"
description: "How to fix ALSA errors and get Claude Code's voice mode working on WSL2 by bridging ALSA to WSLg's PulseAudio server."
tags: ["wsl2", "claude-code", "linux", "audio"]
---

## The Problem

If you try `/voice` in Claude Code on WSL2, you'll likely see a wall of ALSA errors:

```
ALSA lib confmisc.c:855:(parse_card) cannot find card '0'
ALSA lib conf.c:5207:(_snd_config_evaluate) function snd_func_card_inum returned error: No such file or directory
...
ALSA lib pcm.c:2722:(snd_pcm_open_noupdate) Unknown PCM default
```

WSL2 doesn't expose physical sound hardware. There are no ALSA sound cards, so any application that tries to open the default ALSA PCM device fails immediately.

## What WSLg Provides

WSLg (the GUI/multimedia layer in WSL2) actually runs a PulseAudio server that bridges Windows audio over RDP. You can verify this yourself:

```bash
pactl info | grep -E "Server Name|Default Source|Default Sink"
```

You should see `RDPSink` (speakers) and `RDPSource` (microphone). The audio infrastructure exists; the problem is that nothing connects ALSA to it. Applications using PulseAudio natively work fine, but anything going through ALSA (including Claude Code's native audio module) hits a dead end.

## The Fix

Two packages solve this. On Arch Linux:

```bash
sudo pacman -S pulseaudio-alsa alsa-utils
```

On Ubuntu/Debian:

```bash
sudo apt install pulseaudio-utils alsa-utils
```

`pulseaudio-alsa` installs the ALSA PulseAudio plugin and drops config files into `/etc/alsa/conf.d/` that redirect all ALSA calls through PulseAudio. `alsa-utils` provides `arecord`, which matters because of how Claude Code selects its audio backend.

## Why Both Packages Matter

Claude Code tries three recording methods on Linux, in order:

1. A native CPAL/ALSA module (compiled into the binary)
2. `arecord` (from `alsa-utils`)
3. `rec` (from SoX)

The native module talks to ALSA directly through `snd_pcm_open`. In theory, `pulseaudio-alsa` should make this work by routing those calls through PulseAudio. In practice, the native module doesn't always behave well through the PulseAudio ALSA plugin. In my case, installing `pulseaudio-alsa` alone fixed the error messages but voice mode still couldn't capture audio. PulseAudio itself was working (I verified with `parecord`), but the native CPAL module sitting between Claude Code and ALSA wasn't picking it up.

`arecord` works reliably through the same plugin. Installing `alsa-utils` gives Claude Code a working fallback when the native module fails. Without it, Claude Code would need SoX as a last resort, which is a heavier dependency for a single `rec` command.

## Verifying It Works

After installing both packages, you can confirm audio capture is working before trying `/voice` again:

```bash
# Test that arecord captures real audio (not silence)
timeout 3 arecord -f S16_LE -r 16000 -c 1 -t raw -q /tmp/test.raw
python3 -c "
import struct
data = open('/tmp/test.raw', 'rb').read()
samples = struct.unpack(f'<{len(data)//2}h', data)
print(f'Max amplitude: {max(abs(s) for s in samples)} / 32767')
"
```

If the max amplitude is above zero, your microphone is being captured. Make sure your mic is also enabled in Windows sound settings, since WSLg bridges whatever Windows has active.

No reboot or service restart is needed. Run `/voice` in Claude Code and it should work immediately.
