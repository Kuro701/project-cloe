# Jophiel — local image generation and editing

Cloe's fourth skill: local text-to-image generation and text-instructed image editing, running on the same single GPU as everything else. Point it at a prompt and it draws; point it at an existing image and an instruction and it edits — no cloud image API, no per-image cost.

## What it does

One skill, two connected panel modes sharing a single toggle:

- **Quill mode — generation.** Text prompt in, image out. Built on a self-hosted ComfyUI instance running FLUX.1 [schnell] (a quantized GGUF build chosen to fit the VRAM budget alongside everything else Cloe runs).
- **Eye mode — editing.** An existing image plus a text instruction in, an edited image out, via Qwen-Image-Edit. The toggle button itself is the mode indicator — its icon morphs between a quill and an eye as you switch, rather than the mode living in a separate label.
- **Shared skeleton, distinct feel.** Both modes share the same layout — showcase image, a filmstrip history of past generations/edits with an "edit this" hand-off between the two modes, and a prompt bar — but each has its own visual language (soft ink-bloom motion for generation, a slowly rotating iris/scan-line motif for editing) so which mode is active is legible at a glance, not just from a header label.
- **Same VRAM-safety pattern as Metatron.** An explicit engine on/off control rather than an always-resident model, because a second heavy local generation engine competing for the same 12 GB card as the language model and the 3D pipeline needed the same disciplined budget — this skill inherited that design directly rather than re-deriving it.

## Built by porting a proven pattern — and catching what didn't port cleanly

Jophiel's engine lifecycle (spawn, health-probe, stop) was deliberately ported from Metatron's already-proven implementation rather than written fresh. That paid off in speed, but one place the port diverged mattered: Metatron kills its engine's whole process tree (`taskkill /T /F`) because the engine is launched through a `cmd.exe` wrapper that doesn't hand off directly. Jophiel's first version stopped only the wrapper itself, using a plain process-terminate call — which took down the empty wrapper, reported success, and left the actual generation engine running underneath it, still holding VRAM. Fixed by matching Metatron's tree-kill exactly, and verified working after the fix, not just after the first "looks fixed" read of the code.

A second, unrelated fix in the same week: restarting the rest of the Cloe stack wasn't reliably sweeping Jophiel's own process alongside it, for the ordinary reason that process-matching by launch pattern is fragile — the fix hardened the match rather than relying on an incidental catch-all.

## Where it stands

Live and running — the engine has been observed correctly claiming and releasing VRAM through real restarts and manual on/off cycles. Generation mode is proven end to end; the edit mode and cross-session history persistence are active work.
