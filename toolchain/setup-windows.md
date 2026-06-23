# NGPC toolchain — verified Windows / MSYS2 setup

This is a verified setup path for the Toshiba `cc900` NGPC toolchain on Windows, driven from MSYS2 bash. It records the build chain and — more importantly — every PATH, shell and TMPDIR trap that bites between a clean checkout and a working `.ngp` ROM. Each trap below cost real time to diagnose; none of them are obvious from the toolchain's own error messages.

---

## Getting the toolchain (skip the dead Dropbox links)

The standard NGPC starting point is the `ameliandev/ngpc-project-template` GitHub repo. Its README tells you to download the toolchain (`ngpcbins`) from two Dropbox URLs. **Do not.** Those URLs have degraded — following them returns Dropbox's HTML landing page (~185 KB), not the archive, for both `?dl=1` and `?raw=1`.

The same archives ship **directly inside the cloned template**:

- `template/install/ngpcbins_x86_x64.zip` (~68 MB — modern x86-64 Windows/Linux)
- `template/install/ngpcbins_winXP.zip` (~66 MB — legacy)

So the bootstrap is simply: `git clone` the template, unzip `install/ngpcbins_x86_x64.zip`, done. No internet beyond the clone, no Dropbox cooperation. The zip extracts to `ngpcbins/` containing `T900/` (the compiler tree) and `EMUDIR/BizHawk-2.2.1/` (an NGPC-capable emulator, also bundled).

Keep `template/` as an immutable vendor clone — copy what you need into a project folder rather than editing the reference in place.

## Project layout

```
NGPC/
├── template/                       vendor clone (do not edit)
│   └── install/ngpcbins_*.zip      toolchain ships here
├── toolchain/ngpcbins/
│   ├── T900/BIN/                   cc900 tulink tuconv s242ngp asm900 MAKE.EXE
│   ├── T900/INCLUDE/               libc + IO900*.H (TLCS-900H CPU headers)
│   ├── T900/LIB/                   c900ml.lib  c900pic.lib
│   └── EMUDIR/BizHawk-2.2.1/       bundled emulator
└── hello/                          your project (main.c, ngpc.h, Makefile, build.sh, ...)
```

## The build chain

Four stages, each binary invoked in turn:

```sh
cc900 -c -O3 file.c -o file.rel            # each .c → .rel object
tulink -la -o main.abs ngpc.lcf system.lib main.rel library.rel core/rom/flash.rel
tuconv -Fs24 main.abs                      # → main.s24 (Motorola S-Record 24-bit) + .map
s242ngp main.s24                           # → main.ngp ROM
```

Do this **by hand first** on a new toolchain, one binary at a time, before wrapping it in a Makefile — the vintage tools fail in opaque ways, and you want to know which stage broke.

## Required environment

- `THOME=/d/.../toolchain/ngpcbins/T900` — `cc900` reads this to find its includes and libs.
- `$THOME/BIN` must be on `PATH`, in **MSYS2 mount style** (see trap 1 below).

A `build.sh` wrapper that sets `THOME` + `PATH` and invokes the bundled `MAKE.EXE` is the right entrypoint; don't bypass it.

---

## The traps (each one is real and each one is non-obvious)

### Trap 1 — `cc900` needs an MSYS2 `/d/`-style PATH, and needs its BIN dir *on* PATH

`cc900.exe` is not a single binary: it spawns `thc1` (C frontend pass 1) and `thc2` (pass 2) as children, resolving them via `PATH`. Two failure modes:

- Invoke `cc900` by absolute path without adding its dir to `PATH` → `cc900-Fatal-116 : Can't execute ".\thc1"`. The `.\thc1` string is misleading; the fix is to put `$THOME/BIN` on `PATH` and call `cc900.exe` by name.
- Even with the dir on `PATH`, a Windows `D:/...`-style entry makes MSYS2 bash's lookup silently fail (`which cc900.exe` returns empty, though `ls` on the same path works). Use the mount style:

```sh
# ❌ FAILS — which returns not-found, cc900 never runs
export PATH="D:/.../T900/BIN:$PATH"
# ✅ WORKS
export PATH="/d/.../T900/BIN:$PATH"
```

First diagnostic for `cc900-Fatal-116`: run `which cc900.exe`. Empty → wrong PATH style.

### Trap 2 — the bundled GNU Make 3.78 dispatches recipes to `cmd.exe`, not bash

`MAKE.EXE` in `ngpcbins/T900/BIN/` is GNU Make 3.78.1 Win32. It runs recipe lines through the Windows command processor, **not** `/bin/sh`. So Unix tools in recipes fail — `find` resolves to Windows `FIND.EXE`, etc.

Wrap POSIX recipes in `bash -c`:

```makefile
clean:
	@bash -c "find . -name '*.rel' -delete 2>/dev/null; rm -rf $(OUT)"
```

(`SHELL := /usr/bin/bash` at the top works too but is less portable — `bash -c` wrapping is preferred.) If a future session installs a newer Make via pacman this trap disappears, but the bundled one is the zero-dependency default.

### Trap 3 — MSYS2 bash strips `TMPDIR` (bites GCC-based side-builds, not `cc900`)

This one applies when you build *other* things under MSYS2 — for example a libretro emulator core (relevant to the link-cable work). MSYS2's runtime strips `TMPDIR`/`TEMP`/`TMP` on process start, before `/etc/profile` runs, so plain `export TMPDIR=...` and `make -e TMPDIR=...` don't survive. GCC then falls back to `C:\WINDOWS\` and fails:

```
Cannot create temporary file in C:\WINDOWS\: Permission denied
```

The reliable fix is to set the vars *before* the MSYS2 runtime starts, via `env.exe` as the Make SHELL:

```makefile
SHELL := C:/msys64/usr/bin/env.exe \
         TMPDIR=D:/path/.tmp TEMP=D:/path/.tmp TMP=D:/path/.tmp \
         C:/msys64/usr/bin/bash.exe
```

Use forward-slash paths. Verify with `env.exe TMPDIR=X bash -c 'echo $TMPDIR'` → prints `X` (plain `export` prints empty). This is unrelated to trap 2 — different toolchain, different stack.

### Trap 4 — RetroArch pauses background windows

When running two emulators at once (e.g. link-cable testing), RetroArch's default `pause_nonactive = "true"` freezes the out-of-focus instance, which is fatal for a multi-emulator session. Flip it to `"false"` in `retroarch.cfg`.

---

For the binary inventory and versions see [`binaries-inventory.md`](binaries-inventory.md); for the compiler's hard limits and inline-asm gotchas see [`known-issues.md`](known-issues.md).
