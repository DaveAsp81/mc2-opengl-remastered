# Newer Tactic Movement Handling — Draft — 2026-11-06

**Slice:** Propose minimal movement-behavior additions for the five newer tactic enums.  
**Status:** Draft only. No code changes applied.  
**Constraint:** All new behavior must be gated (or only reachable when WEIGHTS_B is on). Stock path unchanged.

## Existing special-cases (baseline)

In `code/warrior.cpp` attack/movement update:

- `TACTIC_STOP_AND_FIRE`: after a successful fire, `clearMoveOrders()`.
- `TACTIC_TURRET`: suppress path recalculation even when out of range or weapons not ready.
- Flanks / Joust: carried in the order bitfield; movement uses the normal pursue / optimal-range path.

The five newer values currently fall through to the default pursue + optimal-range logic.

## Proposed light-touch behaviors

All of the following are intended to live inside the existing movement decision tree, gated by the current tactic value (which is only non-NONE when WEIGHTS_B is on).

### TACTIC_INDIRECT_FIRE (7)
**Intent:** Stay at long range; prefer not to close.
- When computing `desiredAttackRange`, bias toward the vehicle’s long/extreme range instead of optimal.
- Suppress the “close to optimal” recalc if already beyond medium range.
- Still allow pivot-to-target and firing.

### TACTIC_HULL_DOWN (8)
**Intent:** Prefer stationary or minimal movement once a firing position is reached.
- After the first successful fire (or once within optimal range), treat like a softer STOP_AND_FIRE: do not request a new path unless LOS is lost.
- Do not clear move orders on every shot (unlike full stop-and-fire); just refuse new path requests while LOS holds.

### TACTIC_FIGHTING_WITHDRAW (9)
**Intent:** Move away from the target while still able to fire.
- When a path is requested, prefer a goal that increases distance (or use existing withdraw-order machinery if a clean hook exists).
- Do not pursue. Bias `pursue = false` for this tactic even if the order was created with pursue true.
- Allow firing while the withdraw path is active.

### TACTIC_PURSUE (10)
**Intent:** Close aggressively; ignore optimal-range hold.
- Force `recalcPath = true` and `MOVEPARAM_SOMEWHERE_ELSE` whenever distance > short range.
- Suppress the “pivot in place” branch that normally triggers when already at optimal range.
- Higher priority on keeping a path active.

### TACTIC_HIT_AND_RUN (11)
**Intent:** Close, fire, then open distance.
- Track a simple per-tick or per-order phase (can reuse an existing flag or a new `hitAndRunPhase` on the runtime if needed).
- Phase 1 (approach): behave like PURSUE.
- Phase 2 (after successful fire or weapons recycle): behave like FIGHTING_WITHDRAW for a short distance or time, then return to phase 1.
- Minimal implementation: after a fire, request a path that increases distance; after the path completes or a timeout, allow approach again.

## Implementation notes

- Preferred location: the combat-order branch of the movement decision tree in `warrior.cpp` (around the existing `attackParams.tactic` checks).
- Keep each behavior to a few lines so the default path remains the common case.
- Newer tactics must not alter behavior when the tactic field is `TACTIC_NONE` (the stock default).
- If a clean existing withdraw or “somewhere else” path already exists, reuse it rather than inventing new pathfinding.

## Suggested order of implementation

1. PURSUE and INDIRECT_FIRE (simplest range bias).
2. HULL_DOWN (soft stop).
3. FIGHTING_WITHDRAW (distance-increasing goal).
4. HIT_AND_RUN (requires a tiny phase flag).

## Validation (requires Windows build)

- Enable `MC2_TACTIC_WEIGHTS=1` and `MC2_TACTIC_WEIGHTS_B=1`.
- Load a mission with a `[Tactics]` block that heavily weights one new tactic.
- Confirm stock behavior with both gates off.
- Visual / log check that the chosen tactic produces the expected movement (close hard, stay long, hold position, withdraw, or cycle).

## Files that would be touched

- `code/warrior.cpp` (movement decision tree)
- Possibly `code/mech_brain_runtime.h` (only if HIT_AND_RUN needs a phase flag)
- Docs / advisor note once the behaviors land

**Next action:** Implement the two simplest (PURSUE + INDIRECT_FIRE) as a gated patch, or pause for visual-validation plan.
