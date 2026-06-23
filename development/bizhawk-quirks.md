# BizHawk first-launch quirk (NGPC)

A small, empirically-documented quirk that costs time and confusion if you don't know about it. The bundled emulator in the `ngpcbins` archive is `BizHawk-2.2.1`, and it has a reproducible first-launch failure when started in the background from MSYS2 bash.

---

## The symptom

Launching the emulator in the background from MSYS2 bash —

```sh
EmuHawk.exe path/to/rom.ngp &
```

— the **first** invocation:

- returns exit code 0,
- echoes a PID,
- but **does not open a window**.

A second invocation a few seconds later opens the window correctly. This was observed three separate times in a single session (across three different verification steps) before the pattern was recognised.

## Why it matters

The first launch *looks* successful — clean exit code, a PID — so it's easy to report "the emulator is running" when in fact no window opened. That mismatch between the apparent result and the actual state is the real trap.

## Root cause

Not definitively established. Plausible candidates:

- a lockfile / single-instance check in BizHawk 2.2.1 that briefly blocks immediately after a previous instance closed (the prior window may take a few hundred ms to release a ROM-loaded lock);
- `&` from MSYS2 bash detaching the process before window creation completes, if some resource isn't yet free;
- host-specific process-scheduling behaviour.

## Workaround

Just relaunch. The second launch always works. Specifically:

- Do **not** add sleep+retry hacks or change the launch command.
- Do **not** claim "emulator launched" with confidence after a first background launch that may have hit this — wait for visual confirmation that the window is up, or proactively relaunch if confirmation is slow.

It's a two-line note, but it's exactly the kind of thing that silently eats twenty minutes the first time, so it's worth writing down.
