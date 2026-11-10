# GroovyMAME-NLC

GroovyMAME 0.289 (Switchres 2.22f) with three small additions for playing on a
**MiSTer via the Groovy core (nogpu / `-video mister`)**:

1. **Audio fix:** audio on the MiSTer no longer sounds tinny or metallic.
2. **GroovyNLC compression:** `-mister_compression nlc` for the GroovyNLC core.
3. **nonag:** no info or warning screens at startup.

Apart from these three commits, this is GroovyMAME unchanged. If you don't play
on a MiSTer, use the official GroovyMAME instead.

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

---

## Usage

Same as official GroovyMAME. For the MiSTer, for example:

```
mame -video mister -mister_ip 192.168.1.161 -mister_compression nlc ddonpach
```

Or in `mame.ini`:

```
video                     mister
mister_ip                 192.168.1.161
mister_compression        nlc
```

Everything else (`mister_fd_margin`, `mister_interlaced_fb`, Switchres options,
`-sound`, `-audio_latency` …) works exactly as in official GroovyMAME.

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
