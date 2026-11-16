=(CamputersLynx_Senhor notes)=-

Tested: Working Video 720p, 1080p & Sound.

___
# Camputers Lynx for MiSTer

MiSTer core of the **Camputers Lynx** (1983) that brings together the two
Lynx cores by **Kyp** in a single core. The model is chosen in the OSD:

| Model | Kyp's core |
|---|---|
| Lynx 48K, 96K and 96K with Scorpion ROM | [lynx.poseidon](https://github.com/Kyp069/lynx.poseidon) ([lynx](https://github.com/Kyp069/lynx)) |
| Lynx 128K | [lynx128.poseidon](https://github.com/Kyp069/lynx128.poseidon) ([lynx128](https://github.com/Kyp069/lynx128)) |

MiSTer port: Shaeon (Carlos Palmero).

Documentation in Spanish: [README_es.md](README_es.md).

## Installation

- Copy the `.rbf` to `/media/fat/_Computer/`.
- Copy `roms/boot.rom` (48K/96K) and `roms/boot1.rom` (128K) to
  `/media/fat/games/Lynx/`. Both are loaded automatically when the core starts.
- Tapes (`.tzx`) and disks (`.ldf`) can go in the same folder.

See [roms/README.md](roms/README.md) for what each ROM file contains. They are
built from the MAME ROM sets (`lynx48k`, `lynx96k`, `lynx128k`) and include
the disk ROM (XDOS): type `XROM` to start the Lynx DOS.

## OSD

| Option | |
|---|---|
| Mount LDF | Disk image for the WD1793 controller (Lynx DOS, `XROM`) |
| Load TZX | Tape. It is kept in the DDR3, so there is no size limit |
| Model | 96K, 96K Scorpion, 48K, 128K. **Applied on reset** ("Reset & apply model") |
| Bank 2 CAS enable | 48K/96K only (as in Kyp's core) |
| Video | Aspect ratio, scandoubler/HQ2x/scanlines, integer scaling |
| Audio & Tape | Tape sound, invert the audio input, swap joysticks |
| Tape play / stop | Same as PgUp / PgDn |

## Keys

- **F9**: reset.
- **PgUp / PgDn**: tape play / stop.
- Caps Lock = SHIFT LOCK, Esc = ESCAPE, Backspace = DELETE, Ctrl = CONTROL,
  cursor keys = cursor keys. The rest follow the Lynx layout (`matrix.v` in
  Kyp's core).
- Joysticks: port 0x7A (joystick 1) and 0x7B (joystick 2), with one fire button.
- Tape from a real recorder: audio input of the I/O board (ADC).

## How it works

Both machines are inside the core, and only the selected one runs. The other
one is held in reset. They share the SDRAM, the WD1793, the tape, the keyboard,
audio and video.

| | 48K / 96K | 128K |
|---|---|---|
| CPU | 4 MHz | 6 MHz |
| Pixel / 6845 | 6 MHz / 0.75 MHz | 12 MHz / 1.5 MHz |
| Video RAM | 2 × 16K (BRAM) | 64K (BRAM) |
| ROM | `boot.rom`, 64K | `boot1.rom`, 32K |

- **Kyp's cores are git submodules and are not modified** (`rtl/lynx` and
  `rtl/lynx128`). Both have modules called `lynx` and `video` with different
  contents, which Quartus does not allow. Before each build, `rtl/pre_flow.tcl`
  writes a copy of `lynx.v` + `video.v` of each core to `rtl/gen/`, with the
  modules renamed (`lynx48`, `lynx128`) and a `de` output (6845 display
  enable) for MiSTer's blanking. If an upstream change breaks one of the
  patterns, the build stops with an error. The shared modules (T80, 6845,
  WD1793, keyboard) are taken from `rtl/lynx`; they are identical in both
  repositories.
- **ROMs**: Main_MiSTer loads `boot1.rom` with index 0x40 and `boot.rom` with
  index 0. SDRAM map: `boot.rom` at 0x00000, `boot1.rom` at 0x10000, RAM at
  0x20000 (and the 128K's bank 4 at 0x30000).
- **SDRAM**: `rtl/sdram_mister.v`, with the same interface as Kyp's `sdram.v`
  but using the usual MiSTer scheme (SDRAM_CLK = inverted clk_sys, a single
  ACTIVE and READ/WRITE with auto-precharge). It refreshes on its own while
  the CPU is in reset (loading the ROMs).
- **Video RAM**: `rtl/vram.v` replaces Kyp's `dprf`: Quartus 17 cannot map
  `dprf` into the Cyclone V M10K blocks (read-during-write behaviour).
- **Blanking**: horizontal from the 6845 DE, delayed by one character like the
  pixels in `video.v`; vertical from the lines with DE in the previous frame
  (the 6845 is programmed by the ROM).
- **Tape**: `rtl/tape_ddr.sv` and `rtl/tzxplayer.vhd` from
  [ZXP3_MiSTer](https://github.com/shaeon/ZXP3_MiSTer) (DDR3 storage,
  generalized data blocks 0x19, loops).

## Building

```
git clone --recursive https://github.com/shaeon/Lynx_MiSTer.git
```

(`--recursive` is needed: Kyp's cores bring the T80 as a nested submodule.)
Open `Lynx.qpf` with Quartus 17.0 and compile, or run `build.bat`, which
leaves the RBF in `releases/`.

GitHub Actions (`.github/workflows/build.yml`) compiles the core on every push
with Quartus 17.0 (`raetro/quartus:17.0` Docker image) and keeps the `.rbf`,
the reports and the SDRAM timing detail (`tools/rutas_sdram.tcl`) as an
artifact in the *Actions* tab. Pushing a `v*` tag also creates a Release with
the `.rbf`.

To update Kyp's cores: `git -C rtl/lynx pull` / `git -C rtl/lynx128 pull`
and commit the new submodule pointers. Nothing is patched inside them.

## Credits and thanks

- **Kyp** (Kyp069): the Lynx 48K/96K and 128K cores this port is built on.
- **RetroWiki** and its FPGA team (RW-FPGA-devel-Team): rampa069 and ron made
  the first Lynx port for MiSTer (from Kyp's ZX-Uno core) and
  [lynx2wav](https://github.com/RW-FPGA-devel-Team/lynx2wav); and thanks to
  the whole RetroWiki community for keeping the Lynx alive.
- The [Lynx48_MiSTer](https://github.com/MiSTer-devel/Lynx48_MiSTer)
  contributors (rampa069, JasonA, JimmyStones, alanswx).
- MAME (camplynx driver) and the people who dumped the Lynx ROMs; Jynx, PALE
  and the people who documented the Lynx (Pete Todd, w5Mike's ROM analysis).
- T80 (Daniel Wallner, MikeJ and others), UM6845R and WD1793 (MiSTer
  developers, Sorgelig), tzxplayer (György Szombathelyi), and the MiSTer
  framework (MiSTer-devel).

## Licence

GPL v3, like Kyp's cores (the `sys/` framework is GPL v2 or later). The ROMs
are the property of their authors and are included for preservation.

---

Developed with the help of Claude (Anthropic).
