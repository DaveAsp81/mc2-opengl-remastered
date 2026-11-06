# Material Override Hook — Sketch — 2026-11-06

**Slice:** Optional per-mech material override that points at normal/ORM (or a library entry).  
**Status:** Sketch only. Not applied.  
**Goal:** Let a mech opt into modern maps without changing stock behavior.

## Existing pattern to follow

In `mclib/mech3d.cpp` `resetPaintScheme`:

- AO and normal maps are loaded by name helpers (`ImportedMechAoTexName`, `ImportedMechNormalTexName`).
- Load only if `textureOrKtxSidecarExists`.
- Set `aoValid` / `normalValid` only on success.
- At submit time the corresponding `renderFlags` bit is set only if valid.
- Missing file → bit clear → shader no-op.

`[Import] Source=` is already a strict opt-in for GLB geometry (gated by `MC2_ASSIMP_MECH_IMPORT`).

## Proposed hook

### 1. Ini / sidecar field (optional)
In the mech `.ini` (or `.mcasset.json`):

```ini
[Material]
Normal=madcat_normal
Orm=madcat_orm
; or
Library=CorrugatedSteel006A
```

Empty / absent → current albedo-only (or existing AO/normal name helpers) behavior.

### 2. Load path (mirror resetPaintScheme)
```cpp
// MATERIAL-OVERRIDE-1: optional named maps.
// Only attempted when the ini block exists; missing files stay invalid.
if (mechFile.seekBlock("Material") == NO_ERR) {
    char nrmName[256] = "";
    char ormName[256] = "";
    mechFile.readIdString("Normal", nrmName, 255);
    mechFile.readIdString("Orm", ormName, 255);

    if (nrmName[0] && !normalValid) {
        // same path construction + textureOrKtxSidecarExists + loadTexture
        // as the existing NORMALS-1 block
        // on success: normalValid = true
    }
    if (ormName[0] && !ormValid) {
        // same pattern; on success: ormValid = true
        // (requires the ORM sampler from ORM-SAMPLER-RECON)
    }
}
```

### 3. Library shorthand (optional later)
If `Library=CorrugatedSteel006A` is present, resolve to:
- `gameassets/materials/CorrugatedSteel006A/normal_linear.ktx2`
- `gameassets/materials/CorrugatedSteel006A/orm_linear.ktx2`

and load those instead of per-mech names. Still optional; missing library → no-op.

### 4. Submit-time bits
Already specified in the ORM recon:
- `normalValid` → `renderFlags |= 0x20` (existing)
- `ormValid` → `renderFlags |= 0x40` (new)

## Stock degrade
- No `[Material]` block → identical to today.
- Block present but files missing → bits stay clear → shader no-op.
- Works for both stock ASE mechs and `[Import]` GLB mechs.

## Suggested commit message (when applied)
```
feat(mech): optional [Material] override for normal/ORM

Ini block Normal= / Orm= (or Library=) loads maps if present.
Missing files leave the corresponding valid bit clear.
Stock mechs unchanged.
```

## Files
- `mclib/mech3d.cpp` (load + valid flags)
- Mech `.ini` / `.mcasset.json` (optional block)
- Depends on ORM-SAMPLER-RECON for the Orm= path
- This sketch

**Next action:** Apply together with the ORM sampler sketch, or add a Library= resolver once a second material pack exists.
