# The Orb UI

Cloe's desktop presence is a floating orb, not a window with a title bar and a taskbar entry. This document covers how that's built and the design language that ties it together — conceptually, no real markup or code.

## A transparent, always-on-top surface

The overlay is a **Tauri v2** window with true per-pixel transparency: no opaque background rectangle anywhere, so the orb and any open panel appear to float directly on the desktop, over whatever else is running. The window itself covers the full screen (so panels can appear anywhere), which creates the central UI problem: most of that window is empty space, and empty space has to be *click-through* — a click over nothing should reach the desktop app underneath, not get swallowed by an invisible overlay.

The solution is a hit-region model: at any moment, the app knows which screen rectangles correspond to actually-drawn UI (the orb itself, an open panel, a button) and only those regions capture input — everything else passes straight through. This has to be recomputed whenever the orb's state changes (parked in a corner vs. active, a panel open vs. closed), and getting it right across multiple monitors with mismatched DPI scaling was one of the harder pieces of this UI — a stale hit-region after a state change is exactly the kind of bug that looks like "the app stopped responding to clicks" with no error anywhere.

## Mood as a visual language

The orb's glow, color, and animation reflect Cloe's current mood state coming from the backend — she doesn't just report her mood in text, she visibly is in it. Mood transitions animate rather than snap, using a small shared vocabulary of breathing/pulsing motions (slow ease-in-out opacity and scale changes, never anything sharp or jarring) that's reused consistently across the orb itself and every skill panel, rather than each panel inventing its own motion style. Every animated element also respects reduced-motion preferences.

## Skill panels: one visual system, many sub-minds

Each skill sub-mind (see [`ARCHITECTURE.md`](ARCHITECTURE.md)) gets its own panel, but they're built as variations on one shared visual system rather than independent designs:

- A **dedicated color slot** per skill, assigned from a small fixed palette so each sub-mind reads as visually distinct at a glance without the UI turning into a rainbow.
- The **same idle-motion vocabulary** as the core orb (see above) — a new panel borrows the established breathing/pulse language instead of introducing its own.
- A **native mounting pattern**: panels live inside the same overlay window and lifecycle as the orb itself (open/close, drag, click-region registration) rather than spawning as separate OS windows — which keeps every panel behaving consistently under multi-monitor drag and the click-routing model above.
- An **honest empty state** for anything not yet built — a dormant skill socket in the panel list looks unbuilt, not "coming soon," in keeping with the project's broader stance against faked capability.

[`VRETIL.md`](VRETIL.md) walks through this pattern being applied end-to-end for a real panel, including how its color and motion were derived from values already live in the running app rather than picked freehand.

## Where the hit-region model actually broke — twice

The click-through system above sounds simple in principle: register the rectangles that are really drawn, let everything else fall through. In practice, "which rectangles are really drawn" turned out to be a list someone has to keep correct by hand every time a panel changes, and two real bugs came from that list quietly drifting out of sync with what was actually on screen.

**A control that existed but couldn't be touched.** Resize handles on a panel are deliberately drawn a few pixels *outside* the panel's own visible edge, so they don't sit on top of — and steal clicks from — the panel's interior content. But the registered click-through region for that panel was defined as exactly the panel's own drawn box, edge to edge. The handles were, technically, standing in the pass-through zone: a click on a resize handle never reached the panel at all, because the OS-level routing had already decided that pixel belonged to the desktop underneath. The panel looked resizable — the handle was right there, visibly drawn — and simply wasn't, with nothing in any log to say why. The fix was to pad the registered region a small, fixed amount past the panel's own edges on every side, wide enough to cover the handles' overhang, which fixed every panel built on the same pattern in one change rather than one panel at a time.

**A panel that vanished the moment the orb collapsed.** Collapsing the orb to its parked corner is supposed to shrink the clickable area down to just that small circle — except for whichever skill panel is currently open, which needs to stay clickable so a panel can stay open and usable while the orb itself tucks out of the way. That exception lives as a short, explicit list of panel identifiers the collapse logic checks against. When a new skill panel was built, adding its identifier to that list was a manual step — and once, it was simply missed. The panel still rendered normally, could still be dragged, still looked completely functional — until the orb was collapsed, at which point every click on it silently stopped landing, because as far as the click-routing system was concerned that panel no longer existed as a target. Recovery was a one-line fix — add the missing identifier — but finding it meant recognizing the symptom as "this is the same carve-out bug that's been hit before," not a new one.

Both bugs share a shape worth naming: a panel that *renders* correctly is not proof that it's actually reachable, because rendering and click-routing are two separate systems kept in sync by hand. The standing practice that came out of this is to treat "does every interactive element on a new or changed panel actually receive a click" as its own explicit check, done after every change to the click-routing bookkeeping — not inferred from the fact that the pixels look right.

## Multi-monitor behavior

The orb can be dragged across monitor boundaries, parked in a corner, and restored to wherever it last was — including after a monitor is unplugged or a resolution changes, which otherwise risks stranding an always-on-top, taskbar-skipping window somewhere off-screen with no obvious way to get it back. Position state is persisted so a restart doesn't reset her to a default corner.

## Data honesty in the UI

A standing project rule shows up directly in how panels are built: a panel with no live data behind it says so, rather than rendering a plausible-looking placeholder. A stats readout with nothing to show is an empty stats readout — never a number that looks real but isn't.
