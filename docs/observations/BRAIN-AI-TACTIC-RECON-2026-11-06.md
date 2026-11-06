# Brain / AI Tactic Review — 2026-11-06

**Slice:** Unit AI, Brain archetypes, tactic selection, pathfinding hooks.  
**Status:** Recon complete. No code changes.  
**Goal:** Identify concrete, stock-safe improvements that raise tactical quality toward 2026 standards.

## Current architecture (as of fork)

- **Archetypes** (`code/brain_archetype.h`): four presets (InnerSphere.Standard, Clan.Standard, Mercenary.Standard, PlayerControlled). Substring match. Explicit Brain{} switches override.
- **Tactic selection** (`code/brain_tactic_select.*`): pure, deterministic (Wang hash), pilot-modulated (gunnery/aggressiveness/courage).  
  - Gate: `MC2_TACTIC_WEIGHTS` (default OFF).  
  - Even when on, result is **trace-only** — not yet wired into `attackParams.tactic` (comment: TACTIC-WEIGHTS-B pending).
- **Order intent queue** (`code/brain_order_intent.h`): deferred order emission, gated `MC2_BRAIN_INTENT_QUEUE` (default OFF).
- **Special dispatch / task queue / world snapshot**: substantial runtime surface already present.
- **Legacy ABL**: still coexists; must degrade cleanly.

## Gaps relative to “better AI / tactics / pathfinding”

1. **Tactic system is not live**  
   - Weighted selection exists and is pilot-aware, but default-off and not consumed by the attack path.  
   - Highest-leverage next step: wire TACTIC-WEIGHTS-B (consume selected tactic) behind the existing gate, with stock path as fallback.

2. **Archetype doctrine is coarse**  
   - Only four presets. Clan/Mercenary differences are real but limited to radii + two flags.  
   - Opportunity: add more switches (preferred range, flanking bias, focus-fire threshold) that the tactic selector can read.

3. **Pathfinding / movement quality**  
   - No evidence of modern navmesh, cover-aware positioning, or line-of-sight-aware flanking in the brain layer.  
   - Current movement is still largely legacy order-driven.  
   - Safe incremental path: expose “preferred stance / cover bias” as a Brain switch that existing order generators can honor, rather than a full navmesh rewrite.

4. **Help / focus-fire coordination**  
   - Help radii exist. No clear “focus the same target as allies within radius” contract visible in the headers reviewed.  
   - Low-risk addition: a Brain switch that biases target selection toward the current most-threatened or most-attacked ally target.

## Recommended next autonomous slices

1. **TACTIC-WEIGHTS-B-WIRE**  
   - Trace the current `selectedTactic` consumer sites.  
   - Propose the minimal change that makes the weighted pick actually affect `attackParams` when the gate is on.  
   - Keep default-off; stock behavior unchanged.

2. **ARCHETYPE-SWITCH-EXTEND**  
   - Add 2–3 new optional switches (e.g. `preferredRange`, `flankBias`, `focusFire`) to the archetype table and consumer.  
   - Document doctrine mapping (Clan: tighter range, less flank; Mercenary: more aggressive flank/pursue).

3. **HELP-FOCUS-CONTRACT**  
   - Recon how help requests currently choose targets.  
   - Propose a named contract (“allies within attackerHelpRadius prefer the same primary target”) that can be implemented without touching pathfinding.

## Constraints

- Default-off or opt-in for any behavior change. Stock missions must play identically with gates off.
- PlayerControlled archetype must remain inert.
- Prefer pure / header-only additions where possible so they stay unit-testable.
- Do not invent a full navmesh in this slice; stay inside existing order/tactic surfaces.

## Files of record

- `code/brain_archetype.h`
- `code/brain_tactic_select.cpp` / `.h`
- `code/brain_order_intent.h`
- `code/mech_brain_runtime.h`
- `code/brain_special_dispatch.*`
- `code/mission.cpp` (archetype application site)
- `.claude/agents/mc2-brain-ai-expert.md` (new advisor)

**Next action:** Begin TACTIC-WEIGHTS-B-WIRE recon (find consumer sites) or pause for direction.
