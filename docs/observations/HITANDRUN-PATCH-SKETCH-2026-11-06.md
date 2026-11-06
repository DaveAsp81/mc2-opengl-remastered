# HIT_AND_RUN Patch Sketch — 2026-11-06

**Slice:** Concrete insertion points and code fragments for TACTIC_HIT_AND_RUN.  
**Status:** Sketch only. Not applied.  
**Gate:** Only active when `attackParams.tactic == TACTIC_HIT_AND_RUN` (requires `MC2_TACTIC_WEIGHTS_B=1`).

## Why a phase flag

HIT_AND_RUN is a cycle:
1. **Approach** — behave like PURSUE (close aggressively).
2. **Disengage** — after a successful fire, open distance (like a short FIGHTING_WITHDRAW).
3. Return to Approach after a distance threshold or short timeout.

This needs a small piece of per-warrior state. Everything else reuses the existing path-request machinery.

## 1. Runtime state (code/mech_brain_runtime.h)

Add next to `selectedTactic` (mission-ephemeral, default 0):

```cpp
// TACTIC-HITANDRUN-1: 0 = approach, 1 = disengage.
// Only meaningful when selectedTactic == TACTIC_HIT_AND_RUN and WEIGHTS_B is on.
uint8_t          hitAndRunPhase     = 0;       // 0=approach, 1=disengage
uint32_t         hitAndRunPhaseTick = 0;       // frame when phase was entered (optional timeout)
```

(The existing `_pad[1]` can be removed or left; the new fields keep the struct mission-ephemeral.)

## 2. Phase transition on successful fire

In the weapon-fire success path in `code/warrior.cpp` (~line 4252, next to the existing STOP_AND_FIRE clear):

```cpp
if (curTacOrder.attackParams.tactic == TACTIC_HIT_AND_RUN && firedWeapon) {
    MechBrainRuntime* rt = getBrainRuntime();
    if (rt && rt->hitAndRunPhase == 0) {
        rt->hitAndRunPhase = 1;                    // switch to disengage
        rt->hitAndRunPhaseTick = g_mc2FrameCounter; // optional
    }
}
```

## 3. Movement bias (same two sites as previous sketches)

Inside the combat-order range-maintenance block (~4657 and ~4837), after computing `distanceToTarget` / `desiredAttackRange`:

```cpp
TacticType tac = curTacOrder.attackParams.tactic;
MechBrainRuntime* rt = (tac == TACTIC_HIT_AND_RUN) ? getBrainRuntime() : nullptr;
uint8_t phase = (rt) ? rt->hitAndRunPhase : 0;

if (tac == TACTIC_HIT_AND_RUN) {
    if (phase == 0) {
        // Approach — force short range (same idea as PURSUE).
        float shortRange = myVehicle->getOptimalFireRange() * 0.5f;
        if (shortRange > 0.0f)
            desiredAttackRange = shortRange;
    } else {
        // Disengage — prefer to open distance.
        float longRange = myVehicle->getOptimalFireRange() * 1.6f;
        if (longRange > desiredAttackRange)
            desiredAttackRange = longRange;
    }
}

// ... existing attackRangeDelta / recalcDelta calculation ...

bool forceRecalc = false;
if (tac == TACTIC_HIT_AND_RUN) {
    if (phase == 0) {
        // Close while farther than short range.
        if (attackRangeDelta > recalcDelta)
            forceRecalc = true;
    } else {
        // Open distance; also flip back to approach once far enough or after a short timeout.
        float medium = myVehicle->getOptimalFireRange();
        bool farEnough = (distanceToTarget > medium * 1.4f);
        bool timedOut = rt && (g_mc2FrameCounter - rt->hitAndRunPhaseTick > 180); // ~3s at 60 Hz
        if (farEnough || timedOut) {
            if (rt) rt->hitAndRunPhase = 0;   // back to approach
        } else if (distanceToTarget < medium * 1.1f) {
            forceRecalc = true;               // still too close — keep withdrawing
        }
    }
}

// Then fold forceRecalc into the existing recalcPath decision exactly as in the PURSUE sketch.
```

## 4. Away-goal while disengaging

When `phase == 1` and a path is about to be requested (same place as the FIGHTING_WITHDRAW override):

```cpp
if (tac == TACTIC_HIT_AND_RUN && phase == 1 && target) {
    Stuff::Vector3D myPos = myVehicle->getPosition();
    Stuff::Vector3D tgtPos = target->getPosition();
    Stuff::Vector3D away = myPos - tgtPos;
    float len = away.GetLength();
    if (len > 1.0f) {
        away /= len;
        float step = myVehicle->getOptimalFireRange() * 0.5f;
        if (step < 40.0f) step = 40.0f;
        Stuff::Vector3D withdrawGoal = myPos + away * step;
        setMoveGoal(MOVEGOAL_LOCATION, &withdrawGoal);
        // fall through to existing requestMovePath
    }
}
```

## Combined cycle

```text
phase 0 (approach):
  desired range = short
  force close if farther than short
  on successful fire → phase = 1

phase 1 (disengage):
  desired range = long
  request away-vector goal
  when distance > medium * 1.4 or timeout → phase = 0
```

## Safety

- Stock path (`TACTIC_NONE`) is untouched.
- Phase flag is only read/written when the tactic is HIT_AND_RUN.
- No serialization change (mission-ephemeral, same policy as the rest of MechBrainRuntime).
- Both gates remain default-OFF.
- Reuses existing path-request and fire-success paths.

## Suggested commit message (when applied)

```
feat(ai): HIT_AND_RUN approach/disengage cycle

When MC2_TACTIC_WEIGHTS_B=1, TACTIC_HIT_AND_RUN closes aggressively,
switches to a short withdraw after a successful fire, then returns
to approach. Stock path unchanged.
```

## Files

- `code/mech_brain_runtime.h` (two new fields)
- `code/warrior.cpp` (fire-success transition + two movement sites)
- This sketch

**Next action:** Combine all five tactic sketches into one ready-to-apply gated patch, or pause for a Windows build/validation cycle.
