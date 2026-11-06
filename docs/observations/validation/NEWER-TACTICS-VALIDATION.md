# Newer Tactics Validation Checklist — 2026-11-06

**Requires:** Windows build of the gated patch + a mission that loads an `_ai.fit`.

## Enable gates
```
set MC2_TACTIC_WEIGHTS=1
set MC2_TACTIC_WEIGHTS_B=1
set MC2_BRAIN_MISSION_SEED=42
```

## Sample FIT blocks (drop into `<mission>_ai.fit`)

### Pursue-heavy
```
[Tactics]
f Pursue           = 0.80
f StopAndFire      = 0.10
f Joust            = 0.10
```

### Indirect-heavy
```
[Tactics]
f IndirectFire     = 0.80
f HullDown         = 0.10
f StopAndFire      = 0.10
```

### HullDown-heavy
```
[Tactics]
f HullDown         = 0.80
f StopAndFire      = 0.15
f IndirectFire     = 0.05
```

### FightingWithdraw-heavy
```
[Tactics]
f FightingWithdraw = 0.80
f HitAndRun        = 0.10
f StopAndFire      = 0.10
```

### HitAndRun-heavy
```
[Tactics]
f HitAndRun        = 0.80
f Pursue           = 0.10
f FightingWithdraw = 0.10
```

## Checks
1. Gates off → identical to stock (no new movement).
2. Gates on + Pursue-heavy → units close aggressively, rarely hold optimal.
3. Gates on + Indirect-heavy → units stay long, do not close just because a weapon is OOR.
4. Gates on + HullDown-heavy → units stop requesting new paths once LOS is good.
5. Gates on + FightingWithdraw-heavy → units move away from target.
6. Gates on + HitAndRun-heavy → approach, fire, open distance, repeat (phase flips visible in logs if tracing is on).

## Expected logs (if stderr tracing is enabled)
- `[BRAIN_TACTIC_SELECT]` when WEIGHTS is on
- `[BRAIN_TACTIC_B] applied tactic=...` when WEIGHTS_B is on
