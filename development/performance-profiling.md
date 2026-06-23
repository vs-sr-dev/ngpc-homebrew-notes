# Performance profiling — the VBCounter-delta frame-miss HUD

A simple, objective fluidity profiler for the NGPC that turns "does it feel smooth?" into a number on screen. The technique is not NGPC-specific — it ports to **any embedded target with a vsync interrupt and a busy-wait sync**, which is why it's documented as a reusable pattern rather than a one-off.

---

## The problem it solves

"Is the game running at full frame rate under load?" is usually answered subjectively, by watching the screen. That's unreliable: a dropped frame here and there is hard to see, and "feels fine" is not evidence. You want an objective, always-visible signal that a frame was missed.

## The technique

The NGPC has a vertical-blank counter (`VBCounter`) that the hardware increments once per frame, independently of your code. The main loop normally does one unit of work and then waits for the next vsync. If your work fits inside one frame, `VBCounter` advances by exactly 1 across each `WaitVsync()`. If your work overran, it advances by 2 or more — that's a missed frame.

So:

```c
u16 vb_before = VBCounter;     // snapshot before this frame's logic
... do all per-frame work ...
WaitVsync();                   // busy-wait for the next vsync
u16 delta = VBCounter - vb_before;
if (delta > 1) frames_missed += (delta - 1);
```

Render `frames_missed` (or the live delta) to a corner of the screen as a HUD, updated every frame. A persistent counter that only resets on hard console reset gives you a running total across an entire play session — so a single missed frame an hour in is still on the scoreboard.

In the NGPC PoC this was an `MS:XXXX` readout on a system scroll-plane row. Cost: ~5 minutes to build, ~100 bytes of ROM. Payoff: the validation of "smooth under load" went from a subjective impression to an objective `MS:0000` across every gameplay phase, including the busy ones.

## Why it's honest as well as useful

The same HUD also keeps you honest about how hard you're actually pushing the hardware. In the PoC, `MS:0000` held steady — but a back-of-envelope check showed the loop was using **~0.3–0.5% of the per-frame instruction budget** (the TMP95C061 does ~6M instructions/s ≈ 100K instructions per frame at 60 Hz; the loop was ~300–500 instructions). So `MS:0000` proved the architecture *works* at that complexity, not that the hardware was stressed. The counter is a necessary signal, not a sufficient one — and having it on screen makes that distinction visible instead of letting "60 fps" be mistaken for "near the limit".

The real stress tests (8–16 sprites + background scroll + tile redraw + trig math + audio mixing on the main CPU) would be where `MS:` starts to climb. When they do, it's the first number to watch.

## Porting it elsewhere

Any platform with (a) a hardware-incremented frame/scanline counter readable from code and (b) a busy-wait-for-vsync sync point supports this directly: PS2, Dreamcast, Lynx, WonderSwan, etc. Snapshot the counter before logic, read it after the sync, and anything above the expected per-frame increment is a miss. It's one of the cheapest objective instruments you can add to an embedded render loop.
