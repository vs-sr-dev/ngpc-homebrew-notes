# ngpc-homebrew-notes

Technical notes for homebrew development on the **SNK Neo Geo Pocket Color** (1999), and documentation of the **first NGPC↔Dreamcast link-cable bridge ever implemented in emulation**.

This repository is a preservation-research effort. The NGPC has a small but real homebrew scene; what it lacks is a consolidated, verified record of the toolchain's quirks and of the link-cable hardware, which had retail usage in 1999–2000 but has **never** been emulated on either side until now. This repo fills both gaps.

---

## What is the Neo Geo Pocket Color?

The NGPC is SNK's 16-bit handheld, released in 1999 as a competitor to the Game Boy Color. It is a well-built little machine with a loyal following, but it was discontinued early when SNK withdrew from the handheld market.

Hardware essentials:

- **CPU:** Toshiba **TLCS-900/H** (TMP95C061) at ~6.144 MHz — a capable 16-bit CISC core. Audio runs on a separate Z80, so it costs the main CPU nothing.
- **Toolchain:** a **proprietary Toshiba C compiler** (`cc900`) plus a linker/converter chain, redistributed by the homebrew scene since around 2000.
- **Link cable:** a serial UART link between two units, used by a handful of retail games — and, paired with the Dreamcast, by several SNK titles that supported NGPC↔DC communication.

## Why this documentation exists

Two reasons.

**The toolchain is fragmenting.** The standard way to get the NGPC toolchain is via links in an old project template — but the historical Dropbox URLs those point to have degraded (they now resolve to Dropbox's HTML landing page, not the actual archive). The toolchain is also wrapped in several Windows/MSYS2 quirks that are easy to hit and hard to diagnose. This repo records a verified setup path and a complete binary inventory, so future bootstrapping needs no working Dropbox link and no rediscovery of the same traps.

**The link cable had never been emulated.** The NGPC↔NGPC and NGPC↔Dreamcast link cables saw retail use (e.g. SNK vs. Capcom: Match of the Millennium paired with Capcom vs. SNK on Dreamcast), but **no emulator in existence emulated the link-cable UART on either side** — a universal ecosystem gap. This repo documents the recon that mapped every realistic emulator target and the implementation that closed the gap end-to-end, NGPC emulator ↔ Dreamcast emulator, verified on screen with both our own homebrew and retail software.

---

## What's in this repository

| Path | Contents |
|---|---|
| [`toolchain/setup-windows.md`](toolchain/setup-windows.md) | A verified Windows/MSYS2 setup path for the `cc900` toolchain, including the build chain and every PATH/shell/TMPDIR trap. |
| [`toolchain/binaries-inventory.md`](toolchain/binaries-inventory.md) | The full toolchain inventory — `cc900`, `tulink`, `tuconv`, `s242ngp`, bundled GNU Make and BizHawk — with versions and provenance. |
| [`toolchain/known-issues.md`](toolchain/known-issues.md) | Known limits: the 32 KB per-translation-unit object-size cap, the degraded Dropbox links, and the `cc900`/ASM900 inline-assembly gotchas. |
| [`development/bizhawk-quirks.md`](development/bizhawk-quirks.md) | An empirically-documented BizHawk first-launch quirk that wastes time if you don't know about it. |
| [`development/performance-profiling.md`](development/performance-profiling.md) | The VBCounter-delta frame-miss HUD — an objective fluidity profiler that ports to any embedded target with a vsync interrupt. |
| [`link-cable/ngpc-dreamcast-bridge.md`](link-cable/ngpc-dreamcast-bridge.md) | The headline work: emulator recon across all NGPC targets, plus the three-ring NGPC↔NGPC↔Dreamcast bridge implementation, verified with homebrew and retail software. |
| [`lessons/portable-patterns.md`](lessons/portable-patterns.md) | Cross-platform lessons extracted from the sessions — diagnostic-HUD-first debugging, asynchronous AI via stagger init, categorically-distinct visual feedback. |

---

## Verifiable artifacts

The NGPC↔Dreamcast bridge is demonstrated on video, running in real time across two separate emulators:

- **[First working bridge test — two of our own homebrew programs](https://www.youtube.com/watch?v=x2IV3T3Z0oM)** — one running on an NGPC emulator, one on a Dreamcast emulator, open simultaneously, exchanging bytes over the virtual link cable.
- **[Retail-software test](https://www.youtube.com/watch?v=mmxnnkZzu-s)** — two commercial games that supported the hardware link cable, with the bridge visibly working live: the NGPC side sends from the NGPC emulator, the Dreamcast side receives on the Dreamcast emulator.

See [`link-cable/ngpc-dreamcast-bridge.md`](link-cable/ngpc-dreamcast-bridge.md) for the full technical account.

---

## Part of the Silicon Relics project

This is one of three documentation repositories produced as part of **Silicon Relics**, a long-running cross-console preservation and homebrew effort focused on obscure and failed vintage platforms:

- **[hyperscan-homebrew](https://github.com/vs-sr-dev/hyperscan-homebrew)** — Mattel HyperScan / S+Core 7
- **[ngpc-homebrew-notes](https://github.com/vs-sr-dev/ngpc-homebrew-notes)** — SNK Neo Geo Pocket Color / TLCS-900H (this repo)
- **[pippin-homebrew](https://github.com/vs-sr-dev/pippin-homebrew)** — Bandai Pippin @WORLD / DingusPPC networking

GitHub profile: **[@vs-sr-dev](https://github.com/vs-sr-dev)**

---

## Attribution

This research is a human + LLM collaboration. Stating that plainly is the honest thing to do — and it is also the point, because the interesting result here is what the collaboration itself makes possible.

**This work would not exist without Claude as a technical bridge. The human brings domain knowledge and direction; Claude provides implementation across architectures the human could not otherwise approach.** Neither side reaches these results alone. A domain expert does not, by hand, patch an emulator's CPU core to add a bit-level UART, write a TCP↔UDP protocol bridge, and reverse-engineer a retail link-cable protocol by tracing it in a debugger. And a language model, on its own, does not know the NGPC link cable existed, that it had never been emulated, or why bridging it to the Dreamcast is a result worth pursuing.

The split, concretely:

- **Human:** decades of encyclopaedic knowledge of the retro-hardware ecosystem; the judgement to recognise a genuine world-first *because* the field is known; project direction — what to look for, why it matters, where it's worth digging; empirical testing across emulators; and the ground-truth calls.
- **Claude (Anthropic) — Opus 4.7, via Claude Code:** translating that direction into implementation — the emulator-core patches, the bridge software, the protocol recon, and the writing of this documentation.

The human author is a professional technical translator whose field has been heavily affected by AI, and who therefore takes a deliberate position: AI used for **net-new work that displaces no one** — such as reverse-engineering an abandoned console's link cable that nobody was working on — is the case worth defending. The collaboration is named openly for that reason; visibility is the point, not a disclaimer.

Findings were verified empirically; nothing here is asserted from the model's prior knowledge alone.

---

## License

MIT — see [LICENSE](LICENSE). The documentation is original work and may be reused freely with attribution.

The Toshiba `cc900` toolchain itself is proprietary (©Toshiba Corporation) and is **not** redistributed here — this repository documents its use, it does not contain it. See [`toolchain/binaries-inventory.md`](toolchain/binaries-inventory.md) for provenance.
