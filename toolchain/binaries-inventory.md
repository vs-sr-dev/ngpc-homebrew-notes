# NGPC toolchain — binary inventory and provenance

A verified inventory of the `ngpcbins` toolchain as shipped inside `ameliandev/ngpc-project-template` (`install/ngpcbins_x86_x64.zip`). Versions and copyright strings are taken from the binaries themselves. This exists so that the contents of a fragmenting, scene-redistributed toolchain are recorded somewhere durable.

The toolchain is proprietary (Toshiba) plus a couple of vintage scene tools. It is **not** redistributed in this repository — this is documentation of what the archive contains, not the archive itself.

---

## Compiler / linker chain (`T900/BIN/`)

| Binary | Identity | Notes |
|---|---|---|
| `cc900` | TLCS-900 C Compiler (32) **v1.10**, © 2009 Toshiba Corporation | Proprietary. Spawns `thc1`/`thc2` internally via PATH. **Hard limit: a single translation unit's object code must not exceed 32 KB** (`T900/README/compiler.txt`). |
| `tulink` | Toshiba Unified Linker (32) **v2.0p**, © 1993–2009 Toshiba | Links `.rel` objects + libs against the `.lcf` linker script. |
| `tuconv` | Toshiba Unified Object Converter (32) **v1.2n**, © 1993–2008 Toshiba | Used with `-Fs24` to emit Motorola S-Record 24-bit. |
| `asm900` | Toshiba ASM900 assembler | Used for inline asm and startup code. Has its own syntax quirks — see [`known-issues.md`](known-issues.md). |
| `s242ngp` | `s24-2-ngp v1.00 by Dark Fader / BlackThunder` | Vintage scene tool. Converts the S24 record into a `.ngp` ROM. |
| `MAKE.EXE` | GNU Make **3.78.1** for Windows32 (FSF, 1988–1999) | Bundled so bootstrapping needs no separate Make. Dispatches recipes via `cmd.exe` — see setup traps. |

## Includes and libraries (`T900/`)

- `T900/INCLUDE/` — libc headers (`stdio.h`, `stdlib.h`, `string.h`, …) plus `IO900*.H`, the **CPU-generic** TLCS-900H I/O register definitions.
- `T900/LIB/` — `c900ml.lib` (large/medium model) and `c900pic.lib` (position-independent).

## Bundled emulator (`EMUDIR/`)

- `EMUDIR/BizHawk-2.2.1/EmuHawk.exe` — an NGPC-capable build of BizHawk, bundled in the same archive. (Note its first-launch quirk: see [`../development/bizhawk-quirks.md`](../development/bizhawk-quirks.md).)

---

## The NGPC-specific build inputs

These are not toolchain binaries but the per-project files the chain consumes. They come from the template's `hello/` scaffold and are the canonical reference.

### Memory map (from `ngpc.lcf`)

| Region | Range | Access |
|---|---|---|
| `io` | `0x000000`–`0x0000FF` | RW, 256 B |
| `ram` | `0x004000`–`0x006BFF` | RW, 11 KB work RAM |
| `rom` | `0x200000`–`0x3FFFFF` | RX, 2 MB cart ROM |

### Cartridge header (first `0x40` bytes of a `.ngp`, set by `carthdr.h`)

| Offset | Size | Field |
|---|---|---|
| `0x00` | 28 B | ASCII `" LICENSED BY SNK CORPORATION"` (required) |
| `0x1C` | 4 B | far-pointer to the `main` entry |
| `0x20` | 2 B | CartID (little-endian) |
| `0x22` | 2 B | System (LE; `0x1000` = Color mode, CartVer 0) |
| `0x24` | 12 B | CartTitle (ASCII, space-padded — must be exactly 12 chars) |
| `0x30` | 16 B | Reserved (zero-filled) |

### Hardware header

`ngpc.h` (from the template) is the canonical register map — `DISP_CTL0`, `TILE_RAM`, `SPRITE_RAM`, `JOYPAD`, and the BIOS comms vectors `VECT_COM*`. Adopt it as the single source of truth; don't wrap or rename it.

---

## Provenance summary

The Toshiba components (`cc900`, `tulink`, `tuconv`, `asm900`, the libc) are commercial Toshiba TLCS-900 development tools, redistributed within the homebrew scene since around 2000. `s242ngp` is a scene tool by Dark Fader / BlackThunder. GNU Make and BizHawk are free software bundled for convenience. The whole archive has been mirrored inside the project template precisely because its original Dropbox distribution has decayed — which is the reason this inventory is worth keeping.
