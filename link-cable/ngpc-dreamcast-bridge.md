# The NGPC↔Dreamcast link cable — first emulation, end to end

The Neo Geo Pocket Color and the Sega Dreamcast could talk to each other over a serial link cable. Several retail games used it: SNK vs. Capcom: Match of the Millennium on NGPC paired with Capcom vs. SNK on Dreamcast, plus Bikkuriman 2000, Cool Cool Toon, KoF Evolution and others. It also worked NGPC↔NGPC for some titles.

**No emulator in existence emulated the link-cable UART on either side.** This was a universal ecosystem gap — the hardware was real, the retail software was real, but nobody had ever made it work in emulation. This document records the recon that mapped the problem and the implementation that closed it, verified on screen with both our own homebrew and retail games.

Demonstrations:
- **[First working bridge — two of our homebrew programs](https://www.youtube.com/watch?v=x2IV3T3Z0oM)**, one on an NGPC emulator and one on a Dreamcast emulator, running simultaneously and exchanging bytes.
- **[Retail-software test](https://www.youtube.com/watch?v=mmxnnkZzu-s)** — two commercial games that supported the hardware link cable, with the bridge visibly working live (NGPC sends from the NGPC emulator, Dreamcast receives on the Dreamcast emulator).

---

## The shape of the problem

A link-cable bridge needs both endpoints to emit and accept serial bytes. The recon found the two sides in very different states:

- **Dreamcast side (Flycast): already solved, zero patches needed.** Flycast has a `NullModemPipe` (`core/network/null_modem.h`) hooked into the SH4 SCIF serial port (`core/hw/sh4/modules/serial.cpp`). It speaks UDP with a trivial framing: `{'D', byte}` (2 bytes) for data, `{'B'}` (1 byte) for a break — **no handshake, no checksum, no sync** (confirmed by reading the source and by end-to-end test). It's enabled by config (`[network] BattleCable = yes` + `server = host:port`) and binds one UDP socket on `LocalPort` (default 37391). It activates for *any* Dreamcast title launched, not just the target.
- **NGPC side: nothing emulated the UART.** This was the bottleneck.

### NGPC emulator landscape (the recon)

Every realistic NGPC emulator target was mapped and ranked by implementation effort. The key finding is that *none* of them emulate the TLCS-900H serial UART — they stub it:

| Target | State of the serial UART | Estimated effort |
|---|---|---|
| **NeoPop-SDL lineage** (OpenEmu/NeoPop-Core) | Has a `system_comms_read/poll/write` frontend API wired at the **BIOS-HLE** level; BIOS comms vectors are intercepted. Viable *if* target ROMs use the BIOS vectors. | 4–8 h (BIOS-HLE) |
| **MAME** (`TMP95C061`) | UART explicitly stubbed (`// Fake finish sending data`). But a real bit-level UART exists for the same TLCS-900 family (`tmp94c241_serial.cpp`) — a transplant candidate. | 15–25 h |
| **RACE** (libretro) | No MMIO dispatch for the serial range, no BIOS-HLE. | 16–24 h |
| **ares** | `SerialChannel` is ~5 lines (buffer load/store, no shift, no IRQ, no clock). Clean architecture, would use the `Node::Port` pattern. | 20–30 h |
| **BizHawk/Mednafen** | Mednafen has **no UART file at all** — truly missing, not stubbed. | 2–4 weeks (last resort) |

### Choosing the path empirically

Rather than guess BIOS-HLE vs register-level, the retail target was traced in the MAME debugger: breakpoints on eight BIOS comms vectors plus watchpoints on the serial registers, while SNK vs. Capcom: Match of the Millennium did its comms.

**Result: the game uses BIOS comms vectors only** (`VECT_COMCREATEDATA`, `VECT_COMSENDSTATUS`, `VECT_COMONRTS`, `VECT_COMOFFRTS`, `VECT_COMGETDATA`). Nearly all register pokes originate inside the BIOS ROM itself — invisible under an HLE short-circuit. Only two cart-ROM direct pokes were seen (`SC0CR ← 0x09/0x0A`), and those are setup-only and irrelevant when there's no real UART to configure. This confirmed the **NeoPop BIOS-HLE path is sufficient** for the retail target, and the much larger register-level transplant could be deferred.

The transport protocol was also sampled: frames delimited by `0xFF`, a `0x92` handshake/probe byte, e.g. `FF 31 31 00 00 00 00 FF`. The bridge only needs to be byte-transparent — all semantic interpretation happens on the Dreamcast side.

---

## The implementation — three concentric rings

The bridge was built and validated in three rings, each independently verifiable before adding the next.

### Ring 1 — NGPC-only loopback (BIOS-HLE)

A fork of the `beetle-ngp` libretro core, with `system_comms_{read,poll,write}` patched in `mednafen/ngp/system.c` as a 64-byte ring-buffer loopback, exposed to the ROM through a memory-mapped side-channel (status at `0x6BFC`, byte at `0x6BFD`). A small `hello_link` NGPC ROM calls the BIOS vectors directly. **Validated:** TX/RX counters rise symmetrically, the byte field shows the sent value, status cycles correctly.

### Ring 2 — NGPC↔NGPC over TCP

The same core extended with a Winsock2 TCP backend, role-selected at runtime via an env var (`SR3_LINK_ROLE=server|client`, loopback if unset — Ring 1 preserved as a regression). Non-blocking sockets, `SO_REUSEADDR`, lazy accept + client reconnect. The ROM is **unchanged** — the side-channel stays the interface. **Validated:** `send 0x42` on one side → `recv 0x42` on the other, both directions.

### Ring 3 — NGPC↔Dreamcast via a Python bridge + Flycast

The bridge that joins the two worlds. The NGPC core's TCP stream is translated to Flycast's UDP BattleCable framing by a ~150-line stdlib-only Python script:

```
UDP 37391   Flycast bind (LocalPort default)   ← bridge sendto
UDP 42500   bridge bind                         ← Flycast sendto
TCP 42424   bridge listen                       ← beetle-ngp client
```

Mapping: UDP `{'D', byte}` → TCP byte; TCP byte → UDP `{'D', byte}`; UDP `{'B'}` (break) → dropped (the NGPC has no break). Flycast needs **zero source patches** — only a per-session config flip (`BattleCable = yes` + `server = 127.0.0.1:42500`).

On the Dreamcast side, a ~70-line KallistiOS program (`scif_init/read/write` polled + maple controller input) was built to be the homebrew counterpart for the first test. **Validated on screen** (Ring 3, then both videos): symmetric counters increment on both sides, bytes flow both ways, the Dreamcast holds 60 fps with no latency issues.

---

## A couple of corrections worth recording

Recon notes get things slightly wrong; these were corrected against the source and the live system:

- The Flycast config key is literally **`server = host:port`**, not `NetworkServer` (which is only the internal `OptionString` variable name). `LocalPort` defaults to `NaomiNetwork::SERVER_PORT` = **37391**.
- A boot-time diagnostic to expect, not a bug: when Flycast opens a Dreamcast ROM with `BattleCable = yes`, the KallistiOS `dbgio` banner (~10 bytes) fires through SCIF → the pipe → UDP → the bridge *before* the NGPC TCP peer attaches. The bridge logs these as "no peer, byte dropped". Cosmetic.

## Deferred work for real hardware

Everything above is emulator-validated. Before any of it runs on a physical flash cart, two pieces remain (also noted in [`../toolchain/known-issues.md`](../toolchain/known-issues.md)):

- Replace the memory-mapped side-channel with the canonical `SWI 1` + BIOS vector numbers (`0x13` for COMCREATEDATA, `0x14` for COMGETDATA, derived from the BIOS dispatch table).
- Resolve the ASM900 `ld (mem), W` encoding issue, which the side-channel currently works around.

There is also a known `beetle-ngp` `sngSWI` bug (it reads the vector from the wrong register) that will need attention when targeting MAME/ares/real hardware; Rings 1–2 sidestep it by using direct `call <handler>` instead of `SWI`.
