# HULL_DOWN + FIGHTING_WITHDRAW Patch Sketch — 2026-11-06

**Slice:** Concrete insertion points and code fragments for the next two newer tactics.  
**Status:** Sketch only. Not applied.  
**Gate:** Only active when `attackParams.tactic` is non-NONE (`MC2_TACTIC_WEIGHTS_B=1`).

## Target sites

Same two range-maintenance blocks in `code/warrior.cpp` used by the PURSUE / INDIRECT_FIRE sketch:

1. Primary combat-order path — ~line 4657  
2. Secondary / goal-object path — ~line 4837

These tactics are mostly “do not request a new path” or “request a path that increases distance,” so they fit cleanly after the existing `recalcPath` decision and before the final `if (recalcPath && tactic != TURRET)` request.

## Proposed additions

### 1. Soft-stop for HULL_DOWN (after the range-delta check)

Insert immediately after the existing `else if (… OUT_OF_RANGE)` block, still inside the combat-order branch:

```cpp
// NEWER-TACTIC-2: HULL_DOWN — prefer to hold once a firing position is reached.
// Refuse new path requests while LOS holds and we are not badly out of range.
if (tac == TACTIC_HULL_DOWN) {
    bool haveLOS = target ? myVehicle->lineOfSight(target)
                          : myVehicle->lineOfSight(targetPosition);
    bool badlyOut = (weaponsStatusResult == WEAPONS_STATUS_OUT_OF_RANGE) &&
                    (attackRangeDelta > recalcDelta * 3.0f);
    if (haveLOS && !badlyOut) {
        recalcPath = false;          // hold position
        // Do NOT clearMoveOrders() — unlike full STOP_AND_FIRE.
        // Existing path (if any) is allowed to finish; no new one is requested.
    }
}
```

### 2. Distance-increasing goal for FIGHTING_WITHDRAW

Two small pieces:

**A. Force non-pursue (early in the combat-order branch, before the pivot/pursue split):**

```cpp
// NEWER-TACTIC-2: FIGHTING_WITHDRAW never pursues.
if (curTacOrder.attackParams.tactic == TACTIC_FIGHTING_WITHDRAW)
    curTacOrder.attackParams.pursue = false;
```

**B. When a path would be requested, bias the goal away from the target.**  
Inside the final `if (recalcPath && tac != TACTIC_TURRET)` block, before `setMoveGoal` / `requestMovePath`:

```cpp
if (tac == TACTIC_FIGHTING_WITHDRAW && target) {
    // Compute a simple away-vector goal (no new pathfinder).
    Stuff::Vector3D myPos = myVehicle->getPosition();
    Stuff::Vector3D tgtPos = target->getPosition();
    Stuff::Vector3D away = myPos - tgtPos;
    float len = away.GetLength();
    if (len > 1.0f) {
        away /= len;
        float step = myVehicle->getOptimalFireRange() * 0.6f;
        if (step < 50.0f) step = 50.0f;
        Stuff::Vector3D withdrawGoal = myPos + away * step;
        setMoveGoal(MOVEGOAL_LOCATION, &withdrawGoal);
        // Fall through to the existing requestMovePath call.
    }
}
```

(If `Stuff::Vector3D` subtraction / `GetLength` / normalize differ slightly in this codebase, use the equivalent helpers already present in `warrior.cpp` or `mover.cpp`.)

## Combined control flow (conceptual)

```text
compute desiredAttackRange (existing + PURSUE/INDIRECT bias)
compute attackRangeDelta

if HULL_DOWN && haveLOS && !badlyOut:
    recalcPath = false
else if normal range-delta or out-of-range condition:
    recalcPath = true

if FIGHTING_WITHDRAW:
    pursue = false
    if recalcPath:
        override goal to away-vector point

if recalcPath && tactic != TURRET:
    requestMovePath(...)
```

## Safety

- Stock path (`TACTIC_NONE`) is untouched.
- HULL_DOWN only suppresses new path requests; it does not clear existing orders or prevent firing.
- FIGHTING_WITHDRAW reuses the existing path-request machinery; it only changes the goal point.
- No new members required for these two tactics (unlike HIT_AND_RUN).
- Both gates remain default-OFF.

## Accessor / helper note

- Fire ranges: `getMinFireRange()`, `getMaxFireRange()`, `getOptimalFireRange()` exist on `Mover`.
- Position: `getPosition()` is available.
- Vector math: confirm the exact `Stuff::Vector3D` operators in the local headers before applying; the intent is a simple normalized away-step.

## Suggested commit message (when applied)

```
feat(ai): HULL_DOWN hold and FIGHTING_WITHDRAW away-goal

When MC2_TACTIC_WEIGHTS_B=1, HULL_DOWN suppresses new path
requests while LOS holds; FIGHTING_WITHDRAW forces non-pursue
and requests a short away-vector goal. Stock path unchanged.
```

## Files

- `code/warrior.cpp` (two sites + one early pursue override)
- This sketch

**Next action:** Apply this + the PURSUE/INDIRECT sketch together as one gated patch, or draft HIT_AND_RUN (requires a phase flag).
