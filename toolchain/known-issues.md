# NGPC toolchain — known issues and limits

Hard limits and sharp edges in the `cc900` / ASM900 toolchain and its distribution. None of these are showstoppers, but each one will cost you time if you meet it without warning.

---

## The 32 KB per-translation-unit object limit

`cc900` will not produce more than **32 KB of object code from a single translation unit** (`.c` file). This is documented in `T900/README/compiler.txt` and is a hard compiler limit, not a linker one.

The cartridge ROM space is 2 MB, so this is never a total-size problem — it's a *file-granularity* problem. The fix is mundane: split large modules across several `.c` files and let `tulink` combine them. Worth knowing up front so you structure a growing project into compilation units from the start rather than discovering the wall mid-build.

## The Dropbox distribution has degraded

The canonical toolchain download links (two Dropbox URLs in the `ngpc-project-template` README) no longer serve the archives — they resolve to Dropbox's HTML landing page for both `?dl=1` and `?raw=1`. Anyone following the official instructions today will download ~185 KB of HTML and a confusing failure.

The working source is the copy bundled inside the same template repo at `install/ngpcbins_x86_x64.zip`. This is covered in [`setup-windows.md`](setup-windows.md); it is repeated here because "the documented download is broken" is itself a known issue someone will need to find.

---

## `cc900` / ASM900 inline-assembly gotchas

When writing inline asm via `__ASM("...")`, the Toshiba assembler diverges from modern TLCS-900 assembly references in ways it does **not** always diagnose — some wrong forms assemble silently to the wrong opcode. These were learned the hard way while writing BIOS-comms wrappers for the link-cable work.

### Immediates take no `#` prefix

Modern references write `#imm`; ASM900 rejects it with `ASM900-Error-200 Syntax error`.

```
✓  ld   RW3, 0x26
✗  ld   RW3, #0x26
```

### 8-bit register names: use the current-bank short forms

`A`, `B`, `C`, `D`, `E`, `H`, `L`, `W` work and emit current-bank byte accesses (this is what `cc900`'s own codegen uses). Bank-explicit names (`RA3`, `RB3`, `RW3`) are accepted for **immediate** loads but **rejected** as memory-access operands:

```
✓  ld   RW3, 0x26                  # immediate to bank-3 W
✗  ld   RB3, (_g_comms_tx_byte)    # ASM900-Error-230 Operand type mismatch
✗  ld   (_g_comms_rx_status), RW3  # same error
```

### Bare `W` in `ld (mem), W` is ambiguous — silently wrong

`ld A, (_sym)` and `ld (_sym), A` are fine. But `ld (_sym), W` **assembles without error and produces a wrong/no-op opcode** (W collides with 16-bit WA in some contexts). Observed empirically: a BIOS routine wrote bank-3 `W=1`, but reading it back via `ld (_mem), W` yielded 0 every time.

*Workaround:* for a status/return value, use the default `A`, or sidestep registers entirely with a memory-mapped side-channel.

### Symbol naming: C → ASM

`cc900` prefixes C externs with a single underscore in ASM scope: C `g_foo` is `_g_foo` in `__ASM`. Double-underscored labels (e.g. `__startup`) are ASM-local labels referenced verbatim.

### `EQU` works with spaces or tabs

```
__ASM("VECT_FLASHWRITE EQU 6");     // ok
__ASM("VECT_FLASHWRITE\tEQU\t6");   // ok
```

### Always verify what was actually emitted

```sh
cc900.exe -S -O3 src.c -o src.asm
```

Because the assembler accepts some wrong forms silently, dump the `-S` output at the first sign of inline-asm weirdness, and verify any non-standard register transfer with a sentinel: preload the destination with `0x88`, do the load, check whether the value actually changed.

---

## A note on the link cable and real hardware

The link-cable work (see [`../link-cable/ngpc-dreamcast-bridge.md`](../link-cable/ngpc-dreamcast-bridge.md)) was validated against emulators and uses a memory-mapped side-channel plus direct BIOS-vector calls. Before shipping any of it on a **real flash cart**, two known pieces of deferred work apply: switching from the side-channel workaround to canonical `SWI 1` + BIOS vector numbers, and resolving the `ld (mem), W` encoding issue above. Those are tracked in the link-cable document.
