# PURSUE + INDIRECT_FIRE Patch Sketch — 2026-11-06

**Slice:** Concrete insertion points and code fragments for the two simplest newer tactics.  
**Status:** Sketch only. Not applied.  
**Gate:** Only active when `attackParams.tactic` is non-NONE (i.e. `MC2_TACTIC_WEIGHTS_B=1`).

## Target sites

Two nearly identical range-maintenance blocks in `code/warrior.cpp`:

1. Primary combat-order path — around line 4657–4667  
2. Secondary / goal-object path — around line 4837–4847

Both currently do:

```cpp
float desiredAttackRange = myVehicle->getOrderedFireRange();
float attackRangeDelta = distanceToTarget - desiredAttackRange;
float recalcDelta = Terrain::worldUnitsPerVertex * metersPerWorldUnit;
if ((curTacOrder.attackParams.tactic != TACTIC_STOP_AND_FIRE) &&
    (curTacOrder.attackParams.tactic != TACTIC_TURRET) &&
    ((attackRangeDelta > recalcDelta) || (attackRangeDelta < -recalcDelta))) {
    recalcPath = true;
    moveParams = MOVEPARAM_SOMEWHERE_ELSE;
}
else if ((curTacOrder.attackParams.tactic != TACTIC_TURRET) &&
         (weaponsStatusResult == WEAPONS_STATUS_OUT_OF_RANGE)) {
    recalcPath = true;
    moveParams = MOVEPARAM_SOMEWHERE_ELSE;
}
```

## Proposed change (apply to both sites)

Replace the `desiredAttackRange` line and the following `if` with the version below.  
The new logic is a pure local bias; no new members, no new env vars.

```cpp
float desiredAttackRange = myVehicle->getOrderedFireRange();

// NEWER-TACTIC-1: range bias for PURSUE / INDIRECT_FIRE.
// Only non-NONE when MC2_TACTIC_WEIGHTS_B is on; stock path unchanged.
TacticType tac = curTacOrder.attackParams.tactic;
if (tac == TACTIC_PURSUE) {
    // Prefer short range — force close.
    float shortRange = myVehicle->getShortFireRange();   // or getOptimalFireRange() * 0.5f if no short accessor
    if (shortRange > 0.0f)
        desiredAttackRange = shortRange;
}
else if (tac == TACTIC_INDIRECT_FIRE) {
    // Prefer long / extreme range — stay back.
    float longRange = myVehicle->getLongFireRange();     // or getExtremeFireRange() / getOptimalFireRange() * 1.75f
    if (longRange > desiredAttackRange)
        desiredAttackRange = longRange;
}

float attackRangeDelta = distanceToTarget - desiredAttackRange;
float recalcDelta = Terrain::worldUnitsPerVertex * metersPerWorldUnit;

bool forceRecalc = false;
if (tac == TACTIC_PURSUE) {
    // Always try to close while farther than short range.
    if (attackRangeDelta > recalcDelta)
        forceRecalc = true;
}
else if (tac == TACTIC_INDIRECT_FIRE) {
    // Only move if we have drifted inside medium range; otherwise hold.
    float medium = myVehicle->getOptimalFireRange();
    if (distanceToTarget < medium * 0.85f)
        forceRecalc = true;
}

if (forceRecalc ||
    ((tac != TACTIC_STOP_AND_FIRE) && (tac != TACTIC_TURRET) &&
     ((attackRangeDelta > recalcDelta) || (attackRangeDelta < -recalcDelta)))) {
    recalcPath = true;
    moveParams = MOVEPARAM_SOMEWHERE_ELSE;
}
else if ((tac != TACTIC_TURRET) && (weaponsStatusResult == WEAPONS_STATUS_OUT_OF_RANGE)) {
    // INDIRECT_FIRE should not close just because a weapon is out of range
    // if we are already at the biased long range.
    if (tac != TACTIC_INDIRECT_FIRE) {
        recalcPath = true;
        moveParams = MOVEPARAM_SOMEWHERE_ELSE;
    }
}
```

## Accessor note

If `getShortFireRange()` / `getLongFireRange()` do not exist, fall back to:

- PURSUE: `desiredAttackRange *= 0.45f;`
- INDIRECT_FIRE: `desiredAttackRange *= 1.8f;`

(Confirm exact accessors with a quick grep on `Mover` / `BattleMech` before applying.)

## Safety

- When `tactic == TACTIC_NONE` (stock default) the new branches are skipped and the original `if` condition is used unchanged.
- No new state, no serialization impact.
- Both gates remain default-OFF.

## Suggested commit message (when applied)

```
feat(ai): range bias for TACTIC_PURSUE and TACTIC_INDIRECT_FIRE

When MC2_TACTIC_WEIGHTS_B=1, PURSUE biases desired range short and
forces close; INDIRECT_FIRE biases long and suppresses close-to-optimal
recalc. Stock path (tactic NONE) unchanged.
```

## Files

- `code/warrior.cpp` (two sites)
- This sketch

**Next action:** Confirm fire-range accessors, then apply the patch, or move to HULL_DOWN sketch.
