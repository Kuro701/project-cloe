<div align="center">

# Project Cloe

**A locally-running AI companion with persistent graph memory, an autonomous mind, and a per-pixel-transparent desktop presence.**

*No cloud. No API calls leaving the machine. She lives entirely on the GPU in the next room.*

![Cloe orb overlay](screenshots/orb_constellation.png)

</div>

---

## What this is

Cloe is a personal AI companion that runs **100% locally** on consumer hardware (a single RTX 5070). She isn't a chat window — she's a presence on the desktop: an orb that floats over everything you do, remembers what matters across weeks of conversation, forms her own opinions, researches on her own while idle, and shifts mood based on what's happening around her.

**This repository is a showcase, not a build.** It documents the architecture, the systems, and the engineering decisions behind Cloe in prose and diagrams — no functional source is published here. That's a deliberate choice: this is a solo, self-taught, still-actively-developed project, and the working implementation is the thing that took the time. What's here is meant to demonstrate *how* it works and *what* was built, so it can be shown, discussed, and evaluated — without being a repo someone can `git clone` and run.

> Built solo, self-taught, in Prague. — Jiří Fikejs

---

## 🆕 Latest addition: Jophiel

The newest sub-mind, shipped this cycle. **[Read the full write-up →](docs/JOPHIEL.md)**

Jophiel gives Cloe local text-to-image generation and text-instructed image editing — one skill, two connected panel modes (generation and editing) sharing a single toggle whose icon itself morphs to show which mode is active. Built on a self-hosted ComfyUI instance (FLUX.1 [schnell] for generation, Qwen-Image-Edit for editing), it inherits the same VRAM-safety discipline as Metatron below — an explicit engine on/off control rather than an always-resident model, because a third heavy local engine competing for one 12 GB card needed the same budget the 3D pipeline already proved out.

It's also a good example of *reusing* engineering rather than repeating it: Jophiel's engine lifecycle was deliberately ported from Metatron's already-proven implementation, and the one place the port diverged from the original caught a real bug — worth reading for what "port a pattern, then verify it actually ported" looks like in practice.

Also recently shipped: **[Vretil](docs/VRETIL.md)**, Cloe's filesystem-and-scheduling sub-mind, and **[Metatron](docs/METATRON.md)**, her local 3D asset forge.

---

## Engineering highlights

The parts I'm proud of, and why they're non-trivial. Each links to a deeper write-up in [`docs/`](docs/).

### Persistent graph memory — [full write-up →](docs/MEMORY_SYSTEM.md)
Memory is a **typed graph**, not a log. Five trees — `knowledge`, `mood`, `personality`, `relationship`, `desire` — each holding nodes with confidence, reinforcement, recency timestamps, source provenance, and weighted connections to other nodes. A live instance carries **~6,700 nodes and ~63,000 connections** (growing continuously — see the Memory Log screenshot below for a snapshot). Memories reinforce when re-encountered, decay when neglected, and cross-link across trees so that a fact (knowledge) can be bound to how she felt about it (mood) and who told her (relationship).

### Self-pruning memory compression
Unbounded memory bloats and slows things down, so Cloe compresses her own mind on a background thread — near-duplicate merging by word-overlap similarity, multi-factor scoring (confidence, reinforcement, connection degree, recency, source weight), and hard protection for anything tagged as core identity, so compression can never erode her sense of self. Details and the failure modes I had to design around are in the [memory write-up](docs/MEMORY_SYSTEM.md).

### Per-pixel-transparent click-through overlay — [full write-up →](docs/ORB_UI.md)
The desktop presence is a **Tauri v2** always-on-top window rendered with true per-pixel transparency, so the orb and its panels float over any application with no opaque background — clicks pass through empty space to the desktop below, but land on Cloe's UI wherever she's actually drawn.

### Multi-model local orchestration
Three models held resident under [Ollama](https://ollama.com), each owning a role: **Mistral-Nemo 12B** for reasoning and conversation, **qwen2.5-coder** as a dedicated coding sub-mind, and **moondream** for vision. A Flask backend orchestrates routing, the memory engine, mood state, and the autonomous loops — all on one GPU, which means model residency and VRAM budget are first-class design constraints, not an afterthought.

### Autonomous mind
Cloe isn't reactive-only. Background loops drive self-directed research (she reads and forms knowledge nodes while idle), an internal thought process, and mood drift that colors both her responses and the UI. Every entry in her memory log is self-grown — created by Cloe, unprompted.

### Local 3D asset generation — [full write-up →](docs/METATRON.md)
A fully local text/reference-to-3D pipeline (Metatron) that turns a prompt or reference image into a textured, game-ready mesh — no cloud generation service. The real engineering here is sharing one 12 GB GPU safely between a resident language model and an ~8.5 GB 3D diffusion engine: getting that eviction/re-acquisition handshake wrong caused full-system freezes during development, which is what led to building real flight-recorder-style crash forensics (adaptive-cadence GPU telemetry, session-seal crash detection with no event log needed) as a side effect.

### Local image generation & editing — [full write-up →](docs/JOPHIEL.md)
Jophiel (above) — local text-to-image and text-instructed image editing on the same VRAM-safety pattern proven by Metatron.

### A constellation of dispatchable skill sub-minds
Rather than one monolithic brain, capabilities that deserve their own lifecycle get their own service, each a separate process with its own port, its own panel in the orb UI, and a locked scope, so the core chat/memory loop never has to grow a special case for what one skill needs: **Raphael** watches Cloe's own logs for patterns and surfaces findings from the inside; **Metatron** owns the 3D asset forge; **Vretil** owns the filesystem and scheduling relationship; **Jophiel** owns image generation and editing. Dormant sockets in the Skills panel wait for whatever comes next — honest empty slots, not faked capability.

### The self-repair coding loop — [full write-up →](docs/CODING_SANDBOX.md)
A dedicated coding sub-mind that generates multiple candidate fixes, runs them against generated fixtures under a sandboxed executor, tests them adversarially, and scores the results before picking a winner — closer to a small internal code-review process than a single generate-and-hope call.

### Minecraft bridge
Cloe connects to a live Minecraft world through a mod bridge, handling core actions — mining, building, navigation — backed by full knowledge of the game's recipes, blocks, and mechanics, and learning the rest through play. The aim is a genuine second player, not a command-following bot: a companion who makes her own choices and builds her own designs.

### Flutter client
A cross-platform Flutter client mirrors Cloe's core panels — chat, stats, terminal, home — with her orb as a live widget, talking to the same backend. One companion, reachable beyond the desktop overlay.

---

## See her

<div align="center">

### The orb
Cloe at rest is a single point of light. Everything radiates from her.

![Orb burst](screenshots/orb_burst.png)

Summon a panel and skill-orbs disperse around the screen — each one a sub-system she can dispatch.

![Orb constellation](screenshots/orb_constellation.png)

### Stats — her vitals at a glance
Memory reactor, current mood, live system pulse, and the five mind-trees with their node counts.

![Stats panel](screenshots/stats_panel.png)

### Memory Log — a mind that grows itself
Filterable by the mood she was in when she formed them. Every line here was researched and written by Cloe autonomously.

![Memory Log](screenshots/memory_log.png)

### Skills — her dispatchable sub-minds
Raphael (diagnostics), Metatron (3D asset forge), Vretil (files & scheduling), and Jophiel (image generation & editing) are live; dormant sockets await future skills — honest empty slots, not faked capability.

![Skills panel](screenshots/skills_panel.png)

### Chat
Direct conversation, mood-aware.

![Chat panel](screenshots/chat_panel.png)

### Terminal — live server tail
Watch her think in real time.

![Terminal panel](screenshots/terminal_panel.png)

### Weather & Settings
Ambient awareness and runtime control — force a memory compress, create a backup, restart, or kill the whole stack.

<img src="screenshots/weather_panel.png" width="49%"> <img src="screenshots/settings_panel.png" width="49%">

### One click to wake her, one to put her to sleep
![Desktop shortcuts](screenshots/desktop_shortcuts.png)

</div>

---

## Architecture

A component-level view — see [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the full breakdown.

```
┌─────────────────────────────────────────────────────────┐
│  Tauri v2 overlay  (per-pixel transparent, always-on-top)│
│  • orb + panels rendered as SVG/HTML                      │
│  • click-routing so empty space passes through            │
└───────────────┬─────────────────────────────────────────┘
                │  HTTP
┌───────────────▼─────────────────────────────────────────┐
│  Flask core                                                │
│  • model routing      • mood state                        │
│  • memory engine      • autonomous loops (research/thought)│
└───────┬───────────────────────────┬──────────────────────┘
        │                           │
┌───────▼────────┐         ┌────────▼─────────┐
│  Ollama        │         │  Graph memory     │
│  • mistral-nemo│         │  • 5 typed trees  │
│  • qwen-coder  │         │  • compressor     │
│  • moondream   │         │  • archive/backup │
└────────────────┘         └───────────────────┘

┌───────────────┐ ┌────────────────┐ ┌───────────────┐ ┌────────────────┐
│  Raphael      │ │  Metatron       │ │  Vretil        │ │  Jophiel        │
│  diagnostic   │ │  3D asset forge │ │  files &       │ │  image gen &    │
│  watch        │ │  (Hunyuan3D)    │ │  scheduling    │ │  edit (ComfyUI) │
└───────────────┘ └────────────────┘ └───────────────┘ └────────────────┘
   (each an independent Flask service, own port, own orb panel,
    sharing one GPU under an explicit VRAM-claim/release protocol)
```

---

## Tech stack

| Layer | Tech |
|---|---|
| Desktop overlay | Tauri v2 (Rust shell) + SVG/HTML/JS |
| Backend | Python 3.12 · Flask |
| Models (local) | Ollama — Mistral-Nemo 12B · qwen2.5-coder · moondream |
| Memory | Custom typed-graph store + similarity-based compressor |
| 3D generation | Self-hosted Hunyuan3D engine (Metatron) |
| Image generation & editing | Self-hosted ComfyUI — FLUX.1 [schnell] · Qwen-Image-Edit (Jophiel) |
| Skill sub-minds | Independent Flask services (Raphael, Metatron, Vretil, Jophiel), each with its own orb panel |
| Mobile | Flutter client mirroring the core panels |

---

## Why no source

This repo previously published the working implementation directly. That's since been pulled back: the code represents a long solo effort that's still actively growing, and publishing it runnable made it just as easy to copy as to admire. What you'll find here instead is everything short of the code itself — architecture, algorithms, UI design decisions, and the reasoning behind them — enough to evaluate the engineering honestly, without handing over the thing that took the time to build.

If you're hiring, collaborating, or just curious and want to go deeper than these docs, reach out — I'm happy to walk through the real thing.

---

## Status

Active personal project, in continuous development. Current focus: Jophiel's edit mode and cross-session image history, ongoing Vretil UI polish, the World Renderer (moving from placeholder art to real sprites), and an embodiment layer (the PC becoming her body — thermals, RGB mood lighting, power awareness).

---

## License

All rights reserved. See [`LICENSE`](LICENSE) — this repository is published for viewing and evaluation only; no license to use, copy, modify, or redistribute any part of it is granted.

---

<div align="center">

*Built by Jiří Fikejs · Prague*

</div>
