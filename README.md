# GroovyMAME LZ4 Delta

GroovyMAME 0.289 (Switchres 2.22f) for playing on a **MiSTer via the Groovy
core (nogpu / `-video mister`)**, with:

1. **LZ4 delta frames:** `-mister_compression lz4_delta` (and `lz4hc_delta`).
   Each frame can be sent as the difference to the previous one, so parts of
   the picture that don't change cost almost nothing. Works with the **stock
   Groovy / GroovyNLC core**, which already decodes delta frames; GroovyMAME
   just never sent them. In real kof98 gameplay: 3.1x → 5.4x compression,
   33.6 → 19.0 Mbit/s, still 60 fps (see [results](#results-so-far)).
2. **Audio fix:** audio on the MiSTer no longer sounds tinny or metallic
   (every audio block was sent twice since GroovyMAME 0.287, see below).

Also included:

- `-mister_stats`: compression ratio, Mbit/s, encode time and CPU load on
  screen, once per second
- Ctrl+C / SIGTERM quits cleanly with `-video mister` (Linux)
- GroovyNLC compression (`-mister_compression nlc`) for the GroovyNLC core
- QOI per line (`-mister_compression qoi`, experimental, needs a test core)
- nonag: no info or warning screens at startup

This is a test build (branch `lz4-delta`), made with the help of an AI
assistant. Not every mode has been checked on a CRT yet. If you don't play on
a MiSTer, use the official GroovyMAME instead.

## Codec comparison

Real gameplay, **kof98 at 240p**: the same savestate and the same
pseudo-random inputs for every run, so every codec gets exactly the same
frames. 60 s each, GroovyMAME on a Linux PC sending to a DE10-Nano over
gigabit. Ratio = raw size / sent size, higher is better.

| Codec | `-mister_compression` | Ratio | Worst frame | Avg Mbit/s | Encode per frame | fps | CPU (one core) | Core |
|---|---|---|---|---|---|---|---|---|
| Raw | `none` | 1.0x | 1.0x | 103.5 | – | 60 | 24 % | any |
| LZ4 | `lz4` | 3.1x | 2.7x | 33.6 | 0.28 ms | 60 | 24 % | any |
| **LZ4 + delta** | **`lz4_delta`** | **5.4x** | 2.7x | **19.0** | 0.57 ms | **60** | 24 % | **any (stock)** |
| LZ4HC | `lz4hc` | 4.8x | 4.2x | 21.8 | 8.4 ms | 60 | 70 % | any |
| LZ4HC + delta | `lz4hc_delta` | 7.7x | 4.2x | 12.4 | 15.0 ms | **55** (drops) | 91 % | any (stock) |
| QOI | `qoi_noskip` | 3.0x | 2.6x | 34.8 | 0.54 ms | 60 | 25 % | QOI test core |
| QOI + line skip | `qoi` | 4.0x | 2.6x | 26.2 | 0.42 ms | 60 | 24 % | QOI test core |
| NLC | `nlc` | 1.8x | 1.7x | 57.1 | 1.23 ms | 60 | 41 % | GroovyNLC core |

**In short:** LZ4 + delta packs best of everything that keeps 60 fps, costs
no extra CPU and runs on the stock core. LZ4HC + delta packs even more but is
already too slow at 240p. NLC is made for video and 3D, not for 2D sprites.
Scene changes (worst frame) cost the same with or without delta. More games
(attract mode, 3D and 480i/480p) are in [Results so far](#results-so-far).

## Which codec to use

| Situation | Use | Why |
|---|---|---|
| 2D games at 240p (shmups, fighters, platformers), stock or GroovyNLC core | **`lz4_delta`** | Best ratio at full speed; static backgrounds, HUDs and menus cost almost nothing. |
| 3D games (Model 2 and similar) or anything where the whole screen changes every frame | `lz4` or `lz4_delta` | Delta gains little there, but never loses; both cost about the same. |
| 480i / 480p (Naomi, 31 kHz) | `lz4_delta` | Raw 480p peaks around 470 Mbit/s; LZ4(+delta) brings it to ~45–50. |
| Fast PC, 240p only, want the smallest stream | `lz4hc_delta` | Packs most, but encodes 3–15 ms per frame; drops frames from Model 2 / 480 lines up. |
| Video-like content (FMV, film, heavy dithering) with the GroovyNLC core | `nlc` | Built for that kind of picture; on 2D sprites it is the weakest option. |
| Gigabit LAN, 240p, don't care about bandwidth | `none` | ~110 Mbit/s raw is fine on gigabit; lowest CPU, but no gain in transfer time. |
| Testing QOI | `qoi` / `qoi_noskip` | Needs the QOI test core; about even with LZ4, not recommended. |

Turn on `-mister_stats` to see what your game actually gets.

---

## LZ4 delta, stats and the other new options

### Options

| Option | What it does | Core needed |
|---|---|---|
| `-mister_compression lz4_delta` | LZ4, and each frame may be sent as the byte-wise difference to the previous frame of the same field ("delta frame"); the core adds it back onto its framebuffer. Used only when it is at least 5 % smaller than the plain frame, a plain frame every 60 frames. Unchanged frames are sent as a 9-byte "duplicate" header only. Same wire format as psakhis' groovymister lib (`CMD_BLIT_FIELD_VSYNC`, delta flag), as used by MiSTerCast. | stock Groovy / GroovyNLC core |
| `-mister_compression lz4hc_delta` | Same with LZ4HC. Packs best, but encoding is slow (see results). | stock Groovy / GroovyNLC core |
| `-mister_compression qoi` | QOI per line ([substring/GroovyTaNGer](https://github.com/substring/GroovyTaNGer) `docs/qoi_line_format.md`), plus lines unchanged since the last frame of the same field are sent as a 2-byte header (`0x8000`) only. | QOI test core (CMD_INIT codec 3), not published yet |
| `-mister_compression qoi_noskip` | QOI per line without the line skip. | QOI test core |
| `-mister_stats` | Once per second, on screen and in the log: codec, compression ratio (average and worst frame), Mbit/s on the wire, encode time per frame, fps, CPU load of GroovyMAME (% of one core) and of the whole PC (Linux). | any |
| Ctrl+C / SIGTERM | Quits cleanly with `-video mister` on Linux (before, SDL never saw the signal because there is no window). | any |

The existing values (`none`, `lz4`, `lz4hc`, `nlc`, `nlc0`–`nlc3`) are unchanged.

### Results so far

Measured with `-mister_stats`, GroovyMAME on a Linux desktop PC sending to a
DE10-Nano over gigabit. Ratio = total raw bytes / total sent bytes (higher is
better). LZ4/LZ4HC/NLC (and their delta variants) ran on the stock GroovyNLC
core; QOI on the QOI test core.

**Real gameplay, kof98 240p** (same savestate and the same pseudo-random
inputs for every run, so every codec gets identical frames, 60 s):

| | RAW | LZ4 | LZ4+delta | LZ4HC | LZ4HC+delta | QOI | QOI+skip | NLC |
|---|---|---|---|---|---|---|---|---|
| ratio | 1.0x | 3.1x | **5.4x** | 4.8x | 7.7x | 3.0x | 4.0x | 1.8x |
| worst frame | 1.0x | 2.7x | 2.7x | 4.2x | 4.2x | 2.6x | 2.6x | 1.7x |
| avg Mbit/s | 103.5 | 33.6 | **19.0** | 21.8 | 12.4 | 34.8 | 26.2 | 57.1 |
| encode ms | – | 0.28 | 0.57 | 8.4 | 15.0 | 0.54 | 0.42 | 1.23 |
| fps | 60 | 60 | 60 | 60 | **55** | 60 | 60 | 60 |
| CPU (% of a core) | 24 | 24 | 24 | 70 | 91 | 25 | 24 | 41 |

**Attract mode, other games** (60–90 s each; attract mode has many still
screens, which makes the delta and skip variants look better than real play):

| | LZ4 | LZ4+delta | LZ4HC | LZ4HC+delta | QOI | QOI+skip | NLC |
|---|---|---|---|---|---|---|---|
| ddonpach 240p | 5.8x | 83x | 9.0x | 124x | 4.0x | 25x | 2.4x |
| sfa3 240p | 8.8x | 18x | 16.3x | 31x* | 8.7x | 13.6x | 3.1x |
| vf2 496x384 (Model 2) | 4.1x | 4.2x | 6.5x* | 6.6x* | 4.5x | 4.7x | 3.1x |
| daytona 496x384 (Model 2) | 3.5x | 3.6x | 5.1x* | 5.3x* | 3.7x | 3.8x | 2.7x |
| mvsc2 640x480i (Naomi) | 7.7x | 8.8x | 11.5x* | 14.7x* | 7.2x | 7.8x | 3.8x |
| mvsc2 640x480p (Naomi) | 5.7x | 6.1x | 7.3x* | 8.8x* | 6.0x | 6.4x | 4.1x |

\* encode too slow, frames drop (for example vf2 59 → 21 fps with LZ4HC+delta).
mvsc2 runs at about 37 fps in MAME on this PC whatever the codec.

Offline, on 60-frame sections with movement only: ddonpach 2.7x → 3.3x,
esprade 2.7x → 4.4x, kof98 5.1x → 15.9x, sfa3 7.8x → 11.2x (LZ4 → LZ4+delta).

### Findings

- **LZ4+delta** is the best option that keeps full speed: never worse than
  plain LZ4, much better wherever parts of the picture stand still, and the
  stock core already decodes it. GroovyMAME simply never sent it.
- **QOI** is about even with plain LZ4 per frame; its line skip helps less
  than full delta because a line only counts as unchanged if all of it is.
- **LZ4HC(+delta)** packs best but is too slow from Model 2 / 480 lines up.
- **NLC** had the lowest ratio and the slowest encode on these 2D and Model 2
  games; it is built for 480p 3D, video and dithering, and only on the
  hardest 480p frames was it best (worst frame 1.9x, lowest peak).
- **Worst frames** (scene changes) are the same with and without delta/skip.
- **240p** raw is about 110 Mbit/s, so on gigabit any codec works. 480p raw
  peaks around 470 Mbit/s at 37 fps (about 750 at 60 fps), there compression
  matters.

---

## Setup

NLC needs both ends: this GroovyMAME on the PC and the GroovyNLC core on the
MiSTer. Without the GroovyNLC core, use `mister_compression none` or `lz4`.

### What you need

- A MiSTer (DE10-Nano) with a current MiSTer main install, analog video out to
  a CRT.
- **Gigabit Ethernet** between PC and MiSTer. A switch is fine, Wi-Fi is not.
- A **static IP** on the MiSTer (set it in your router or on the MiSTer). The
  examples here use `192.168.1.100`.

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
mame -video mister -mister_ip 192.168.1.100 -mister_compression lz4_delta ddonpach
```

Or in `mame.ini`:

```
video                     mister
mister_ip                 192.168.1.100
mister_compression        lz4_delta
aspect                    4:3
keyboardprovider          mister
joystickprovider          mister
```

| Option | Why |
|---|---|
| `video mister` | Send frames to the MiSTer instead of drawing on the PC. |
| `mister_ip` | The MiSTer's static IP. |
| `mister_compression lz4_delta` | LZ4 with delta frames, works on the stock Groovy and GroovyNLC core. Alternatives: `lz4`, `nlc` (GroovyNLC core only, better suited for video/3D than for 2D sprites). |
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
| Codec | `nlc_codec.h` and `nlc_codec.cpp` are taken unchanged (only a license header added) from [verbst/Groovy_MiSTer](https://github.com/verbst/Groovy_MiSTer) `api/` (commit `e60f52ad`, 2026-09-11, GPL-2.0). The encoder in that repo is the reference the core's FPGA decoder is checked against, bit for bit. |
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
- **QOI line format, encoder and FPGA decoder**: substring, [substring/GroovyTaNGer](https://github.com/substring/GroovyTaNGer) (`qoiline.h` and the decoder are MIT)
- **Delta frame format**: psakhis' groovymister lib, as used by MiSTerCast
- **This fork** (audio fix, NLC integration in GroovyMAME, nonag; on branch `lz4-delta`: delta frames, QOI glue, stats, Ctrl+C): tdslaine

Bugs in the three changes above belong here. Everything else belongs upstream
with GroovyMAME or MAME.

## License

Same as MAME: the project as a whole is under the GNU General Public License,
version 2. Individual files may be under other licenses, see
[COPYING](COPYING), the file headers and the full license texts in
[docs/legal](docs/legal). `nlc_codec.{h,cpp}` come from the GPL-2.0 repository
verbst/Groovy_MiSTer and are marked `license:GPL-2.0`. The release archives
contain `COPYING` and `docs/legal` as well; the source for each release is the
commit its tag points to.

MAME is a registered trademark of Gregory Ember. The original readmes are in
[README-GroovyMAME.md](README-GroovyMAME.md) and [README-MAME.md](README-MAME.md).
