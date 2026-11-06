# TACTIC-WEIGHTS-B Wire Recon — 2026-11-06

**Slice:** Locate consumer sites for `selectedTactic` / `attackParams.tactic` and assess wiring status.  
**Status:** Recon complete. Wiring already partially present; header comments are stale.  
**Outcome:** Minimal gate-protected path exists. Remaining work is completeness + documentation, not green-field wiring.

## What already exists

### Population (TACTIC-WEIGHTS)
- `code/warrior.cpp` ~2337–2371: when `MC2_TACTIC_WEIGHTS=1`, `selectTacticForWarrior` runs each tick and writes `brainRuntime->selectedTactic`.
- `code/mission.cpp` ~3463 / ~3631: loads `[Tactics]` block from `<mission>_ai.fit` into `tacticWeights[]` (silent if missing).

### Consumption (TACTIC-WEIGHTS-B)
- `code/brain_special_dispatch.cpp`:
  - Gate helper `s_tacticWeightsBEnabled()` (env `MC2_TACTIC_WEIGHTS_B`, default OFF).
  - Two attack-order construction sites (direct apply ~1423 and intent-commit ~1915) write  
    `attackOrder.attackParams.tactic = (TacticType)runtime->selectedTactic` when the gate is on.
- Needs **both** gates: WEIGHTS populates the value, WEIGHTS_B consumes it. With B off the field stays `TACTIC_NONE` (byte-identical to stock).

### Downstream use
- `code/warrior.cpp` movement/attack update checks `curTacOrder.attackParams.tactic` for:
  - Stop-and-fire / turret special cases
  - Range recalculation and path-recalc suppression
- `code/tacordr.cpp` packs the tactic into the order bitfield and uses it for debug strings.

## Stale documentation
- `code/mech_brain_runtime.h:116` still says “TRACE-ONLY: NOT wired into attackParams.tactic (deferred to TACTIC-WEIGHTS-B)”.  
  This is no longer accurate for the special-dispatch path.

## Gaps / remaining work

1. **Coverage of attack-order paths**  
   - Confirm every path that builds a `TACTICAL_ORDER_ATTACK_OBJECT` goes through the two sites above (or add the same two-line gate to any missed path).  
   - Legacy ABL-driven orders may still bypass the brain runtime entirely (expected; stock path).

2. **Newer tactic enums**  
   - Enum includes `TACTIC_INDIRECT_FIRE`, `HULL_DOWN`, `FIGHTING_WITHDRAW`, `PURSUE`, `HIT_AND_RUN`.  
   - Movement logic in `warrior.cpp` only special-cases the original set (STOP_AND_FIRE, TURRET, flanks).  
   - The newer tactics currently fall through to default movement. They need explicit handling if they are to change behavior.

3. **Header / comment hygiene**  
   - Update `mech_brain_runtime.h` comment to reflect that WEIGHTS_B is implemented on the special-dispatch path.

## Minimal next implementation slice (still autonomous)

- Update the stale comment in `mech_brain_runtime.h`.
- Add a short “how to enable” note (both env vars) to the Brain AI advisor or a docs/ file.
- (Optional) Add the same two-line gate to any additional attack-order construction site discovered by a full grep of `TACTICAL_ORDER_ATTACK_OBJECT`.

## Constraints
- Both gates remain default-OFF. Stock behavior unchanged.
- PlayerControlled and non-Enhanced brains are unaffected.

**Files of record:**  
`code/brain_special_dispatch.cpp`, `code/warrior.cpp`, `code/mech_brain_runtime.h`, `code/mission.cpp`, `code/tacordr.cpp`

**Next action:** Apply the comment fix + enablement note, or move to ARCHETYPE-SWITCH-EXTEND recon.
