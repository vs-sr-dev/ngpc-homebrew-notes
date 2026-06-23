# Portable patterns from the NGPC sessions

A handful of working methods that emerged from NGPC development and generalise well beyond it. They are recorded here because they are reusable on any embedded or small-systems target, not because they are NGPC-specific.

---

## Diagnostic HUD first, speculation second

**Rule:** when an input doesn't respond or an output doesn't appear on an embedded target, do **not** start patching ROM code on a hypothesis. Build an on-screen HUD that dumps the raw register / byte / state, then sweep the inputs and watch what actually arrives.

**Why:** in one NGPC session, the A button didn't trigger a field→battle transition, while movement (the same `JOYPAD` register, different bits) worked. There were five plausible causes — bitmask bug, wrong polarity, broken edge-trigger, missing emulator binding, debounce — and blindly patching for each could have cost ~30 minutes per guess. Instead, a one-line HUD (`PrintHex` of the `JOYPAD` register every frame) plus asking the user to press every button revealed in under 60 seconds: direction bits 0–3 work, face-button bits 4–6 are permanently 0. The ROM was correct; the problem was an emulator-side missing pad mapping — entirely outside the ROM code's reach.

**How to apply:**
- *When:* an input doesn't respond, or an output doesn't appear. (Not for rendering bugs — those need a screenshot, not a register dump.)
- *What to render:* the raw byte/register nearest the problem (`JOYPAD`, an RX buffer, a timer counter). Hex is enough.
- *Where:* a screen area free of gameplay, on a system text/scroll layer so it doesn't disturb sprites.
- *Update:* every frame, inside the main loop — not in an event handler.
- *Sweep protocol:* have the user exercise one input at a time and report values; a mapping table emerges in 30–60 seconds.

This generalises to any embedded I/O bring-up: PS2 SIO2 pad registers, Dreamcast maple-bus state, Lynx buttons, a comm RX buffer. The anti-pattern is patching ROM logic before you've dumped the state — it pins your suspicion to the wrong side (ROM vs emulator vs hardware) and burns build/test cycles.

## Asynchronous behaviour via stagger-init

When you have N agents running the *same* logic (enemies, particles, blinking lights), initialising their timers identically makes them move in lockstep — visually robotic. Offsetting each one's start by a small per-index amount (`timer = base + i * step`) produces naturally "irregular tension" with no extra code. In the NGPC combat PoC, `ai_timer = 30 + i*20` was enough to turn three identical-AI enemies from a synchronised block into something that read as independently alive. Stagger is a *feature*, not a trick — it applies to any multi-agent system sharing one update function.

## Visual feedback must be categorically distinct for opposite events

If two game states mean opposite things, their on-screen feedback must differ *in kind*, not just in degree. In the PoC, "enemy winding up to attack" used an X-jitter telegraph; "enemy got hit" originally used a slightly larger X-jitter. They were nearly indistinguishable — the same gesture at different amplitudes reads as the same event. Replacing the hit feedback with a full-sprite blink (sprite-off every 2 frames) made the two instantly legible. The lesson: pick orthogonal feedback channels (position-jitter vs visibility-blink vs colour) for semantically opposite events, not two points on the same axis.

## Escape the local-analysis loop when stuck

After roughly 30 minutes of circumscribed debugging without progress, stop descending into the same code. Offer (or take) one of three escape moves:

1. **Observational reframe** — step up a level: what's changing *above* the thing you're staring at?
2. **Methodology switch** — build a test rig and let the human sweep it, rather than reasoning in your head.
3. **Parameter sweep** — enumerate N concrete values and try them, instead of trying to derive the one right value.

This was codified after it resolved two separate stalls (an NGPC link-cable hang and an unrelated raycaster bug). Long circular debugging is a signal to change *method*, not to debug harder.

## Work manually before writing the Makefile

On vintage toolchains, invoke each binary by hand — one stage at a time — before wrapping the chain in a Makefile. The tools fail in opaque ways, and a four-stage build (`cc900` → `tulink` → `tuconv` → `s242ngp`) is much easier to debug when you can see exactly which stage broke and on which input. Automate only once the chain is known-good by hand.
