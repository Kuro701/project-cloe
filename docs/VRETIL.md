# Vretil — files and scheduling

Named for the angel-scribe of 2 Enoch — *"who writes down all the deeds of the Lord… and reads out the heavenly books."* Vretil is the skill that gives Cloe a standing, always-on relationship with your filesystem: browsing, searching, reading, summarizing, writing, and organizing files, plus a full scheduling and reminders system — all conversational, all running continuously rather than scoped to a single session.

Vretil shipped before Jophiel (Cloe's current newest addition — see [`JOPHIEL.md`](JOPHIEL.md)), and is a good example of how a feature moves from idea to running code here.

## What it does

- **Conversational file access.** Ask her to find, open, read, or summarize something, in plain language — a lightweight intent gate catches file-related phrasing before it ever reaches a model call, so ordinary chat has zero added cost.
- **Create and edit, on explicit instruction.** Vretil resolves what you're referring to against a registry of previously-named files before deciding what to do — a genuinely new thing gets created fresh, while something matching an existing record is either appended to or rewritten, depending on whether the new content reads as a fresh log entry or a correction to something already there.
- **A passive save offer** — after a reply, one lightweight check asks whether what was just said reads as a concrete fact worth keeping, versus ordinary conversation. This is importance-gated, not timer-gated, and it won't re-offer the same fact twice in one session after a decline.
- **Scheduling and reminders** — one-off ("remind me in 20 minutes…"), recurring ("every day at 10pm…"), weekday-scoped, and simple condition-flavored checks, plus listing and cancelling. Stored as a small durable file re-read at boot, so reminders survive a restart.
- **Self-organizing storage.** Anything Vretil creates lands in a small set of flat, top-level category folders rather than an ever-deepening tree. A lightweight classification step checks a new item against a registry of existing categories (each with its own description) before deciding whether it belongs in one of them or needs a brand-new category — new categories are always created at the top level, never nested inside an existing one, which is what keeps the whole structure flat by construction instead of by discipline.

## Design process

Before anything was built, this went through a locked design pass that:

- **Read the real, current state of the system** rather than assuming it — including the actual model-routing table already in use elsewhere in the app, to decide whether Vretil needed its own dedicated model (it didn't: it routes to the same conversational model already warm for chat, avoiding an unnecessary VRAM swap on a single-GPU setup).
- **Explicitly scoped what v1 would and wouldn't do** — file access and scheduling in; shell execution, GUI automation, and external connectors out — locked as a table before implementation started, not discovered mid-build.
- **Defined a permission model up front**: free roam across most of the filesystem, but an explicit yes/no confirmation — stated in plain language, every single time, never cached — before touching the live application's own source code or anything under the system drive. There's no session-level grant to fall back on; every entry into a sensitive location is asked for individually.
- **Derived its visual identity from the real running app, not a guess.** Its assigned color slot was pulled directly from a color value already present — but dormant — in the orb's real source, matching the existing pattern of one reserved slot per skill. Its idle motion reuses the exact breathing/pulse animation vocabulary already shipping elsewhere in the UI (see [`ORB_UI.md`](ORB_UI.md)) rather than inventing a new style for one panel.
- **Went through a genuine collaborative visual design pass** before a single line of UI code was written — the panel takes the form of a book (chat on the left page, a table of contents of tracked projects on the right), with its own hand-drawn sigil built at the real scale and construction of the existing orb art, so it sits convincingly next to the other skill panels rather than looking bolted on.

## Bugs that only showed up once it was real

A handful of integration bugs surfaced once Vretil was actually running alongside the rest of the live system rather than in isolation — the kind of thing that only shows up on real hardware, not in review. Two are worth naming because they're part of a recurring pattern documented in more depth in [`ORB_UI.md`](ORB_UI.md): a missing entry in the list of panels that should stay clickable while the orb is collapsed (fixed by adding it), and resize handles that were visually present but sat just outside the panel's registered click-through region (fixed by padding that region). Both were one-line fixes once correctly diagnosed, and both were caught because clicking-through every interactive element after a UI change is now treated as its own explicit step, not inferred from the panel looking right.

## Where it stands

Vretil is built and running — visually confirmed on real hardware, including drag-and-drop and its native mounting inside the orb overlay (matching the pattern in [`ORB_UI.md`](ORB_UI.md) rather than opening as a separate window). Functional testing of the conversational triggers, the scheduling round-trip, and the file-organization logic is the next step before it's considered fully proven out.
