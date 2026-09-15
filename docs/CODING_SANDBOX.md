# The self-repair coding loop

Cloe has a dedicated coding sub-mind, routed to its own model (`qwen2.5-coder`), used both for conversational coding help and for self-repair tasks against her own systems. This is a summary of the pipeline's shape — not the implementation.

## Generate, test, score — not generate-and-hope

A single generated fix is a coin flip. Instead, the loop:

1. **Generates multiple candidate fixes** for a given problem rather than committing to the first output.
2. **Builds a disposable test fixture** — a small, representative reproduction of the scenario the fix needs to handle — rather than testing against production state.
3. **Runs each candidate inside a sandboxed executor** with restricted filesystem and process access, so a candidate that misbehaves can't affect anything outside its fixture.
4. **Tests adversarially** — beyond the "normal" case, a hostile variant probes whether the fix actually holds up rather than just passing the obvious test.
5. **Scores and compares candidates** on the results, including a content-loss check that flags a candidate that quietly destroyed data it wasn't supposed to touch.
6. **Repairs and retries** when nothing passes cleanly, feeding the failure back in as context for another attempt, up to a bounded number of rounds.

This is closer to a small internal code-review process — generate, test, critique, retry — than a single call trusted at face value.

## Why sandboxing matters here

Letting a model-generated candidate run at all means treating its output as untrusted code, not a natural-language reply. The sandbox exists specifically so a bad candidate — one that writes somewhere it shouldn't, deletes something, or tries to reach the network — fails loudly and safely inside its fixture instead of touching anything real. That boundary has been through structured internal review, including checks for gaps in exactly what it does and doesn't restrict; hardening it is treated as ongoing work rather than a one-time guarantee, the same way any sandbox around untrusted, model-generated code has to be.

## What a real audit found: one guard, four ways around it

The sandbox's write guard worked by intercepting the one function most code uses to write a file. That held up fine against normal candidates — until a structured audit stopped reading the guard and started testing it directly, trying every route a candidate could plausibly take to put bytes on disk instead of just the obvious one.

Four of those routes got through untouched. The guard patched a single name, but the same underlying write capability was reachable through at least four sibling entry points that all resolve to the same lower-level operation — deleting and renaming were genuinely blocked, but writing wasn't, for anything that reached the disk by a slightly different door. The worst case wasn't a candidate misbehaving loudly; it was a candidate that could silently overwrite or truncate a real file outside the sandbox while the run's own bookkeeping still reported a clean pass — the exact failure mode a sandbox exists to prevent, quietly not happening.

The fix was to close every one of those doors the same way the first one was closed, then re-run the identical probe suite against the patched guard: every escape route blocked, and ordinary writes *inside* the sandbox still worked exactly as before. The lesson that stuck past this one bug: a guard around a capability isn't verified by reading which function it wraps — it's verified by trying every other way to reach the same effect, and treating anything still open as the bug, not an edge case.

## Verified answers over confident-sounding ones

A recurring theme in how this pipeline is built: a check that *could* pass silently is worse than one that fails loudly. An unverified result is required to read as unverified in the pipeline's own bookkeeping — never as a quiet pass — because the entire point of the adversarial and scoring steps is to catch the case where something looks fine and isn't.
