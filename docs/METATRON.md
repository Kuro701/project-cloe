# Metatron — the local 3D asset forge

Cloe's third skill: a fully local text/reference-to-3D prop pipeline running entirely on the same single consumer GPU as everything else. Point it at a description or a reference image and it produces a textured, game-ready 3D asset — no cloud generation service, no per-asset API cost.

Named for the angel traditionally described as the "recording angel" — fitting for the sub-mind that turns an idea into a physical (virtual) object.

## What it does

- **Text or reference-image prompt in, textured GLB out.** A studio panel inside the orb: type a prompt or drop a reference, watch the forge queue, preview the result in an offline WebGL altar you can drag-orbit before deciding, then approve, regenerate, or discard.
- **Configurable polygon budget.** A poly-count control on the panel drives the same request end to end — verified against a 100,000-face ask coming back as a 100,000-face mesh, not an engine default silently overriding it.
- **Honest state, always.** Four status lights (service / engine / VRAM / focus) reflect real probes against the running process, not assumed state — if the engine is cold, the panel says so rather than pretending it's ready.

## The hard problem: sharing one GPU between a language model and a 3D diffusion engine

This is the actual engineering content of this skill, and it took several real iterations to get right.

The 3D engine (self-hosted, ~8.5 GB resident) and the conversational model (~7–8 GB resident) cannot both fit on a 12 GB card. Early versions got this wrong in ways that **took the whole PC down** — not the app, the machine — because eviction and re-acquisition of VRAM raced each other:

- **Eviction has to happen before the engine even starts loading**, not concurrently with it — an ordering bug that let the engine spend up to 12 minutes warming *while* the language model still held the card caused a hard crash with no BSOD, no dump, and no diagnostic trail, because the failure mode was a full system freeze, not a clean process crash.
- **A background keepalive thread meant to keep the chat model warm was fighting the eviction it­self** — reloading the 7 GB model onto the card every 60 seconds during the exact window the 3D engine needed it clear. Fixed with an explicit VRAM-claim/release protocol so any thread that would normally re-warm the language model checks first whether another skill currently owns the card.
- **"How much VRAM does the engine actually need" turned out to be unanswerable from outside the process** — a CUDA caching allocator expands to fill whatever is free, so reading usage at any single moment measures the trough or the peak of a load curve, never a stable "need." The fix that stuck: stop trying to predict a number and instead record real outcomes (how much room a job was given, whether it finished, how low free memory actually got) and only ever raise the safety floor on proven failure — never lower it on a single success.

## Crash and freeze forensics — a side effect that became its own subsystem

Chasing the VRAM incidents above with no event-log trace to work from (a full freeze writes almost nothing to Windows' own logs) led to building real diagnostic tooling from scratch: a flight recorder sampling GPU/CPU/RAM telemetry at an adaptive cadence (relaxed when idle, every 2 seconds under pressure), session-seal crash detection that can tell "the process died with the machine" from "the process was killed cleanly" with no event log at all, and a ranked root-cause analyzer that reports its own confidence honestly (hardware-confirmed → telemetry-supported → circumstantial → no-data) rather than guessing with false certainty.

## Where it stands

Live and verified end to end on real hardware, including a 535-sample flight-recorder run confirming the fix: the language model held zero VRAM for eighteen unbroken minutes through a full engine boot and forge cycle, versus roughly fifty seconds before the fix. Manual engine on/off is by design — a cold boot costs several minutes, so it's a deliberate choice to pay that once per work session rather than once per job.
