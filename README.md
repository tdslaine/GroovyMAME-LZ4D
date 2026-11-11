# GroovyMAME-NLC

GroovyMAME 0.289 (Switchres 2.22f) with three small additions for playing on a
**MiSTer via the Groovy core (nogpu / `-video mister`)**:

1. **Audio fix:** audio on the MiSTer no longer sounds tinny or metallic.
2. **GroovyNLC compression:** `-mister_compression nlc` for the GroovyNLC core.
3. **nonag:** no info or warning screens at startup.

Apart from these three commits, this is GroovyMAME unchanged. If you don't play
on a MiSTer, use the official GroovyMAME instead.

---

## Setup

NLC needs both ends: this GroovyMAME on the PC and the GroovyNLC core on the
MiSTer. Without the GroovyNLC core, use `mister_compression none` or `lz4`.

### What you need

- A MiSTer (DE10-Nano) with a current MiSTer main install, analog video out to
  a CRT.
- **Gigabit Ethernet** between PC and MiSTer. A switch is fine, Wi-Fi is not.
- A **static IP** on the MiSTer (set it in your router or on the MiSTer). The
  examples here use `192.168.1.161`.

### 1. MiSTer: install the GroovyNLC core

Use the GroovyNLC release the codec in this fork comes from:
**v1.4** (2026-09-11) from
[verbst/Groovy_MiSTer releases](https://github.com/verbst/Groovy_MiSTer/releases).
The core and the `MiSTer_groovyNLC` binary belong together, always update both.

1. Copy `GroovyNLC_*.rbf` to `/media/fat/_Utility/` and `MiSTer_groovyNLC` to
   `/media/fat/`. With FTP, use binary mode.
2. Add this to `/media/fat/MiSTer.ini`:

   ```
   [GroovyNLC]
   main=MiSTer_groovyNLC
   ```

   The section name must be `GroovyNLC` (the core name, not the `.rbf` file
   name). The official Groovy core can stay installed next to it with its own
   `[Groovy]` section. Settings you had under `[Groovy]` don't apply to
   GroovyNLC; move them or use `[Groovy*]`, which matches both.
3. Start **GroovyNLC** from `_Utility`. A bouncing logo means it is waiting for
   an emulator. It disappears when frames arrive.

The core listens on UDP **32100** (video, audio, control) and UDP **32101**
(inputs). Allow both if the PC has a firewall.

### 2. MiSTer: core settings (OSD)

Open the OSD in the core (F12 or the OSD button) and check:

| OSD option | Set to | Why |
|---|---|---|
| Video > Volatile framebuffer | **Off** | Required for NLC. With On, the picture freezes or goes black after the first frame. |
| Video > Framebuffer lead | 1 | Default. Only raise it if the picture smears under load. |
| Audio > Audio | On | Otherwise no sound from GroovyMAME. |
| Audio > Desired buffer (ms) | 0 | Lowest latency. Raise only if sound breaks up. |
| Server > Joysticks | Analog | Pads on the MiSTer are sent to the PC. |
| Server > PS2 | Keyboard | Only if you want to use a keyboard plugged into the MiSTer (see below). |
| Server > Jumbo frames | Off | Only turn on if the PC network adapter is also set to MTU 3800. |

Then pick **Save settings**, otherwise the core comes back with defaults next
time. Server options are read at core start, so reload the core after changing
them.

### 3. PC: GroovyMAME

On the command line:

```
mame -video mister -mister_ip 192.168.1.161 -mister_compression nlc ddonpach
```

Or in `mame.ini` (this is the setup we play with):

```
video                     mister
mister_ip                 192.168.1.161
mister_compression        nlc
aspect                    4:3
keyboardprovider          mister
joystickprovider          mister
```

| Option | Why |
|---|---|
| `video mister` | Send frames to the MiSTer instead of drawing on the PC. |
| `mister_ip` | The MiSTer's static IP. |
| `mister_compression nlc` | GroovyNLC near level 1. Use `nlc0` for lossless if the network allows it, `nlc2`/`nlc3` only if frames still don't get through. |
| `aspect 4:3` | Without it GroovyMAME takes the aspect of the PC desktop (16:9) and the picture on the CRT is squeezed. |
| `keyboardprovider mister`, `joystickprovider mister` | Use the pads (and keyboard) plugged into the MiSTer. Leave on `auto` to use pads on the PC. |

Everything else (`mister_fd_margin`, `mister_interlaced_fb`, Switchres options,
`-sound`, `-audio_latency` …) works exactly as in official GroovyMAME. Vertical
games: set the rotation in GroovyMAME (`-rol`/`-ror`) or the CRT's orientation,
the core only rotates on its scaler output.

### 4. Check that NLC is active

- Start a game with `-verbose`. The log must show
  `nogpu: compression algorithm nlc, near level 1`. If it says
  `not supported`, the value of `mister_compression` is misspelled.
- On the MiSTer, turn on `Verbose` on the Server page and look at
  `/tmp/groovynlc.log`.

### Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Core stays on the logo | Nothing connected. Check `mister_ip`, the network and the firewall (UDP 32100/32101). |
| Picture freezes or goes black right after the first frame | Volatile framebuffer is On. Set it Off, save, reload the core. |
| Garbage picture | The core can't decode the stream: official Groovy core, or `.rbf` and `MiSTer_groovyNLC` from different releases. Install both from the same GroovyNLC release, or use `mister_compression none`. |
| Picture squeezed on the CRT | Set `aspect 4:3`. |
| Sound tinny or metallic | You are not running a build with the audio fix (see change 1). Check with the WAV test. |
| No sound | Audio is Off in the core OSD. |
| Pads/keyboard on the MiSTer do nothing | `joystickprovider`/`keyboardprovider` not set to `mister`, or Joysticks/PS2 Off in the core. |
| Display taken over by another program | The core serves one client at a time. Close the other emulator. |
| GroovyMAME doesn't quit on Ctrl+C/SIGTERM in mister mode | Known GroovyMAME behaviour. Quit from the game (Esc) or use `-seconds_to_run N` in scripts. `kill -9` can leave the core hanging, then reload the core. |

---

## What changed and why

All changes are on branch `main` as three commits on top of the official
`groovymame0289` branch (`953db38f`, release `gm0289sr222f`). Each is
self-contained and can be cherry-picked on its own.

### 1. Audio blocks were sent twice to the MiSTer

| | |
|---|---|
| Files | `src/emu/sound.cpp`, `src/osd/modules/render/drawnogpu.cpp` |
| Symptom | With `-video mister`, audio on the MiSTer sounds tinny or metallic. Audio on the PC is fine. RetroArch on the same MiSTer is fine. |
| Since | GroovyMAME 0.287, commit `834b5554` "Synchronize audio update with VBlank". 0.283 is not affected. |
| Cause | `screen_device::vblank_end()` also calls `sound().update(0, 0)`. If no speaker produced new samples since the previous update, `m_record_samples` still holds the old count, so `streams_update()` passes the same `m_record_buffer` to the MiSTer (and the WAV writer) again. Every audio block arrives twice. |
| Fix | Reset `m_record_samples` after the buffer has been passed on, and don't send empty audio blocks to the MiSTer. |
| Upstream | Submitted as [antonioginer/GroovyMAME#19](https://github.com/antonioginer/GroovyMAME/pull/19). |

You can check any build for the bug yourself:

```
mame <game> -seconds_to_run 60 -wavwrite test.wav
```

If the bug is present, the WAV is about 120 s long and every block appears
twice. With the fix it is 60 s.

### 2. GroovyNLC compression (`-mister_compression nlc`)

| | |
|---|---|
| Files | `src/osd/modules/render/drawnogpu.cpp`, `src/osd/modules/render/nlc_codec.{h,cpp}`, `src/osd/modules/lib/osdobj_common.cpp`, `scripts/src/osd/modules.lua` |
| What | Near-lossless video codec for the nogpu link: RGB888 to YCoCg-R, MED prediction, Rice packing, 16 px tiles. Frames are smaller than raw, so less data goes over the network to the MiSTer. |
| Needs | The **GroovyNLC core** on the MiSTer ([verbst/Groovy_MiSTer](https://github.com/verbst/Groovy_MiSTer)). The official Groovy core can't decode NLC. It ignores the extra bits in `CMD_INIT` and receives raw frames, so nothing breaks, but you get no compression. |
| Codec | `nlc_codec.h` and `nlc_codec.cpp` are copied unchanged from [verbst/Groovy_MiSTer](https://github.com/verbst/Groovy_MiSTer) `api/` (commit `e60f52ad`, 2026-09-11, GPL-2.0). The encoder in that repo is the reference the core's FPGA decoder is checked against, bit for bit. |
| Glue | Option parsing, the `CMD_INIT` compression byte (codec 2, near level, YCoCg, decode mode 2, Rice) and the call to `nlc_encode()` in `nogpu_blit()`. The bit layout follows verbst's RetroArch GroovyNLC fork, so the core sees the same data from both. If a frame doesn't fit, it is sent raw for that frame. |

Values:

| `-mister_compression` | Meaning |
|---|---|
| `none` | raw (default) |
| `lz4`, `lz4hc` | LZ4, as in official GroovyMAME |
| `nlc` | GroovyNLC, near level 1 (recommended) |
| `nlc0` | GroovyNLC, lossless |
| `nlc1` to `nlc3` | GroovyNLC, near level 1 to 3 (higher = smaller, less exact) |

### 3. nonag: skip startup screens

| | |
|---|---|
| File | `src/frontend/mame/ui/ui.cpp` |
| What | The system information and emulation warning screens are never shown. Games start straight into gameplay. |
| Why | On a cabinet, or on a MiSTer without a keyboard, these screens need a keypress that is often not at hand. |
| vs. `skip_gameinfo` | MAME's own `-skip_gameinfo` only skips the system information screen. The warning screen ("This system has imperfect graphics/sound …") still waits for a key. There is no option for that, which is what this patch adds. |

---

## Tested

Tested with a MiSTer DE10-Nano on a 15 kHz CRT, using the GroovyNLC core:

- **Linux** (CachyOS, GCC 16, PipeWire): ddonpach, ddpsdoj and others. NLC runs
  at full speed, blit about 1 ms. Audio is clean, a 60 s run gives a 60 s WAV.
- **Windows** (mingw-w64 cross build): only tested under Wine so far
  (`-listfull`, `-showusage` and a 20 s headless run). Not yet tested on real
  Windows with a MiSTer.

## Building

Exactly like GroovyMAME or MAME. No extra dependencies.

**Linux**

```
make -j$(nproc) NOWERROR=1 OPTIMIZE=3 REGENIE=1
strip -o groovymame mame
```

**Windows x64 (cross build from Linux with mingw-w64)**

The MiSTer input code uses SDL scancodes, so the SDL2 headers are needed at
build time (SDL2-devel-2.30.12-mingw from libsdl-org). The SDL DLL is not
needed at runtime.

```
make -j$(nproc) TARGETOS=windows OSD=windows PTR64=1 CROSS_BUILD=1 NOWERROR=1 OPTIMIZE=3 REGENIE=1 \
  OVERRIDE_CC=x86_64-w64-mingw32-gcc OVERRIDE_CXX=x86_64-w64-mingw32-g++ \
  OVERRIDE_LD=x86_64-w64-mingw32-g++ OVERRIDE_AR=x86_64-w64-mingw32-ar \
  MINGW64=/usr/x86_64-w64-mingw32 ARCHOPTS=-I<sdl2>/x86_64-w64-mingw32/include
x86_64-w64-mingw32-strip -o groovymame.exe mame.exe
```

`ARCHOPTS` only takes effect together with `REGENIE=1`. A native Windows build
with MSYS2, as described in the MAME docs, should also work but is untested.

## Downloads

Ready-made Linux and Windows x64 builds are on the
[Releases](../../releases) page, built from exactly this source.

## Who and where from

- **MAME**: the MAME team, [mamedev/mame](https://github.com/mamedev/mame)
- **GroovyMAME, Switchres, nogpu**: Calamity (Antonio Giner) and
  contributors, [antonioginer/GroovyMAME](https://github.com/antonioginer/GroovyMAME).
  The emusync audio/video sync is by intealls and Calamity.
- **Groovy MiSTer core**: psakhis, [psakhis/Groovy_MiSTer](https://github.com/psakhis/Groovy_MiSTer)
- **GroovyNLC codec and core**: verbst, [verbst/Groovy_MiSTer](https://github.com/verbst/Groovy_MiSTer)
- **This fork** (audio fix, NLC integration in GroovyMAME, nonag): tdslaine

Bugs in the three changes above belong here. Everything else belongs upstream
with GroovyMAME or MAME.

## License

Same as MAME: the project as a whole is under the GNU General Public License,
version 2 or later. Individual files may be under other licenses, see
[COPYING](COPYING) and the file headers. `nlc_codec.{h,cpp}` come from a
GPL-2.0 repository.

MAME is a registered trademark of Gregory Ember. The original MAME readme is in
[README-MAME.md](README-MAME.md).
