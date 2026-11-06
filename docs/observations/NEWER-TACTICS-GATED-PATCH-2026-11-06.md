# Newer Tactics — Ready-to-Apply Gated Patch — 2026-11-06

**Slice:** Single coherent patch that adds movement behavior for the five newer tactic enums.  
**Status:** Ready to apply. Not yet applied.  
**Gate:** All new behavior is inert unless `MC2_TACTIC_WEIGHTS=1` **and** `MC2_TACTIC_WEIGHTS_B=1`.  
Stock path (`tactic == TACTIC_NONE`) is byte-identical.

## Files touched

- `code/mech_brain_runtime.h` — two new fields (HIT_AND_RUN phase)
- `code/warrior.cpp` — fire-success transition + two movement decision sites

## 1. Runtime state (`code/mech_brain_runtime.h`)

Add immediately after the existing `selectedTactic` declaration:

```cpp
    // TACTIC-HITANDRUN-1: 0 = approach, 1 = disengage.
    // Only meaningful when selectedTactic == TACTIC_HIT_AND_RUN and WEIGHTS_B is on.
    uint8_t          hitAndRunPhase     = 0;       // 0=approach, 1=disengage
    uint32_t         hitAndRunPhaseTick = 0;       // frame when phase was entered
```

## 2. Fire-success phase transition (`code/warrior.cpp`)

Locate the existing block (~line 4252):

```cpp
	if (curTacOrder.attackParams.tactic == TACTIC_STOP_AND_FIRE) {
		if (firedWeapon)
			clearMoveOrders();
	}
```

Insert immediately after it:

```cpp
	if (curTacOrder.attackParams.tactic == TACTIC_HIT_AND_RUN && firedWeapon) {
		MechBrainRuntime* rt = getBrainRuntime();
		if (rt && rt->hitAndRunPhase == 0) {
			rt->hitAndRunPhase = 1;
			rt->hitAndRunPhaseTick = g_mc2FrameCounter;
		}
	}
```

## 3. Movement decision sites (`code/warrior.cpp`)

There are two nearly identical range-maintenance blocks (primary ~4657, secondary ~4837).  
Apply the same change to both.

### 3a. Range bias + forceRecalc (replace the existing desiredAttackRange + if)

Replace from `float desiredAttackRange = ...` through the end of the out-of-range `else if` with:

```cpp
				float distanceToTarget = myVehicle->distanceFrom(targetPosition);
				float desiredAttackRange = myVehicle->getOrderedFireRange();

				// NEWER-TACTICS-1: range bias (only when WEIGHTS_B has set a non-NONE tactic).
				TacticType tac = curTacOrder.attackParams.tactic;
				MechBrainRuntime* rtHR = nullptr;
				uint8_t hrPhase = 0;
				if (tac == TACTIC_HIT_AND_RUN) {
					rtHR = getBrainRuntime();
					if (rtHR) hrPhase = rtHR->hitAndRunPhase;
				}

				if (tac == TACTIC_PURSUE || (tac == TACTIC_HIT_AND_RUN && hrPhase == 0)) {
					float shortRange = myVehicle->getOptimalFireRange() * 0.5f;
					if (shortRange > 0.0f)
						desiredAttackRange = shortRange;
				}
				else if (tac == TACTIC_INDIRECT_FIRE || (tac == TACTIC_HIT_AND_RUN && hrPhase == 1)) {
					float longRange = myVehicle->getOptimalFireRange() * 1.75f;
					if (longRange > desiredAttackRange)
						desiredAttackRange = longRange;
				}

				float attackRangeDelta = distanceToTarget - desiredAttackRange;
				float recalcDelta = Terrain::worldUnitsPerVertex * metersPerWorldUnit;

				bool forceRecalc = false;
				if (tac == TACTIC_PURSUE || (tac == TACTIC_HIT_AND_RUN && hrPhase == 0)) {
					if (attackRangeDelta > recalcDelta)
						forceRecalc = true;
				}
				else if (tac == TACTIC_INDIRECT_FIRE) {
					float medium = myVehicle->getOptimalFireRange();
					if (distanceToTarget < medium * 0.85f)
						forceRecalc = true;
				}
				else if (tac == TACTIC_HIT_AND_RUN && hrPhase == 1) {
					float medium = myVehicle->getOptimalFireRange();
					bool farEnough = (distanceToTarget > medium * 1.4f);
					bool timedOut = rtHR && (g_mc2FrameCounter - rtHR->hitAndRunPhaseTick > 180);
					if (farEnough || timedOut) {
						if (rtHR) rtHR->hitAndRunPhase = 0;
					} else if (distanceToTarget < medium * 1.1f) {
						forceRecalc = true;
					}
				}

				if (forceRecalc ||
				    ((tac != TACTIC_STOP_AND_FIRE) && (tac != TACTIC_TURRET) &&
				     ((attackRangeDelta > recalcDelta) || (attackRangeDelta < -recalcDelta)))) {
					recalcPath = true;
					moveParams = MOVEPARAM_SOMEWHERE_ELSE;
				}
				else if ((tac != TACTIC_TURRET) && (weaponsStatusResult == WEAPONS_STATUS_OUT_OF_RANGE)) {
					if (tac != TACTIC_INDIRECT_FIRE) {
						recalcPath = true;
						moveParams = MOVEPARAM_SOMEWHERE_ELSE;
					}
				}
```

### 3b. HULL_DOWN soft hold

Immediately after the block above (still inside the combat-order branch), add:

```cpp
				// NEWER-TACTICS-1: HULL_DOWN — hold while LOS is good.
				if (tac == TACTIC_HULL_DOWN) {
					bool haveLOS = target ? myVehicle->lineOfSight(target)
					                      : myVehicle->lineOfSight(targetPosition);
					bool badlyOut = (weaponsStatusResult == WEAPONS_STATUS_OUT_OF_RANGE) &&
					                (attackRangeDelta > recalcDelta * 3.0f);
					if (haveLOS && !badlyOut)
						recalcPath = false;
				}
```

### 3c. FIGHTING_WITHDRAW / HIT_AND_RUN away-goal

Early in the combat-order branch (before the pivot/pursue split), add:

```cpp
			if (curTacOrder.attackParams.tactic == TACTIC_FIGHTING_WITHDRAW)
				curTacOrder.attackParams.pursue = false;
```

Then, inside the final `if (recalcPath && tac != TACTIC_TURRET)` block, before `setMoveGoal` / `requestMovePath`:

```cpp
					if ((tac == TACTIC_FIGHTING_WITHDRAW ||
					     (tac == TACTIC_HIT_AND_RUN && hrPhase == 1)) && target) {
						Stuff::Vector3D myPos = myVehicle->getPosition();
						Stuff::Vector3D tgtPos = target->getPosition();
						Stuff::Vector3D away = myPos - tgtPos;
						float len = away.GetLength();
						if (len > 1.0f) {
							away /= len;
							float step = myVehicle->getOptimalFireRange() * 0.55f;
							if (step < 45.0f) step = 45.0f;
							Stuff::Vector3D withdrawGoal = myPos + away * step;
							setMoveGoal(MOVEGOAL_LOCATION, &withdrawGoal);
						}
					}
```

## Behavior summary

| Tactic              | Range bias          | Path behavior                          |
|---------------------|---------------------|----------------------------------------|
| PURSUE              | short (×0.5)        | force close                            |
| INDIRECT_FIRE       | long (×1.75)        | suppress close; hold if already long   |
| HULL_DOWN           | (unchanged)         | refuse new paths while LOS holds       |
| FIGHTING_WITHDRAW   | (unchanged)         | pursue=false + away-vector goal        |
| HIT_AND_RUN         | phase-dependent     | approach (short) ↔ disengage (away)    |

## Safety checklist

- [ ] Both gates default OFF.
- [ ] `tactic == TACTIC_NONE` takes the original code path.
- [ ] No new serialization.
- [ ] HIT_AND_RUN phase is only touched when that tactic is active.
- [ ] Existing STOP_AND_FIRE / TURRET special cases preserved.
- [ ] Vector math uses existing `Stuff::Vector3D` operators (confirm `GetLength` / normalize before landing).

## Suggested commit message

```
feat(ai): gated movement behaviors for newer tactics

When MC2_TACTIC_WEIGHTS=1 and MC2_TACTIC_WEIGHTS_B=1:
- PURSUE closes aggressively
- INDIRECT_FIRE holds long range
- HULL_DOWN holds position while LOS is good
- FIGHTING_WITHDRAW opens distance
- HIT_AND_RUN cycles approach → fire → disengage

Stock path (tactic NONE) unchanged.
```

## Validation (requires Windows build)

1. Both gates off → identical to stock.
2. Both gates on + a mission `_ai.fit` that weights one tactic heavily.
3. Log / visual check for each of the five behaviors.
4. Confirm HIT_AND_RUN phase flips on fire and returns after distance or timeout.

**Ready to apply.** Next step is a Windows build + the validation steps above.
