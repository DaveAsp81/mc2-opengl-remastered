# ORM Sampler Recon — 2026-11-06

**Slice:** Sketch adding a mech ORM sampler that mirrors the existing normal-map path.  
**Status:** Sketch only. Not applied.  
**Goal:** Optional roughness/metal/AO for imported mechs without breaking stock.

## Existing pattern (to mirror)

### Shader (`shaders/mech.frag` + `mech.vert`)
- `v_importedMech` bits (from `renderFlags >> 3`):
  - bit0 = imported
  - bit1 = HAS_AO (renderFlags bit 4)
  - bit2 = HAS_NORMAL (renderFlags bit 5)
- Normal sampling is gated by `(v_importedMech & 4u)` and only occurs when the CPU has set the bit (i.e. the .ktx2 loaded).
- Sampler: `u_mechNormalTex` (unit 7), strength `u_normalStrength`.
- `textureLod(..., 0.0)` used to avoid AMD mip issues.

### CPU (`mclib/mech3d.cpp` ~2921–2947)
- `desc.slot7TexHandle = normalValid ? normalTextureHandle : 0`
- `if (normalValid) desc.renderFlags |= 0x20u` (bit 5)

## Proposed ORM addition

### Bit allocation
- Next free: `v_importedMech` bit3 = HAS_ORM  
- Corresponding `renderFlags` bit 6 (`0x40u`)

### Shader changes (`shaders/mech.frag`)
```glsl
#ifdef MC2_IMPORTED_MECH_MATERIAL
uniform sampler2D u_mechOrmTex;     // unit 8: packed ORM (R=AO, G=Roughness, B=Metal)
uniform float     u_ormStrength;    // 0 = no-op, 1 = full
#endif

// After normal perturbation, before lighting:
#ifdef MC2_IMPORTED_MECH_MATERIAL
if ((v_importedMech & 8u) != 0u) {          // HAS_ORM
    vec3 orm = textureLod(u_mechOrmTex, v_uv, 0.0).rgb;
    float ao   = mix(1.0, orm.r, u_ormStrength);
    float rough= mix(0.5, orm.g, u_ormStrength);   // 0.5 = neutral fallback
    float metal= orm.b * u_ormStrength;
    // Apply ao to albedo (or combine with existing AO path)
    c.rgb *= ao;
    // rough / metal consumed by the StandardLit / GGX path when enabled;
    // when StandardLit is off they are ignored (no behavior change).
}
#endif
```

### Vertex flag widen (`shaders/mech.vert`)
```glsl
// bit0=imported, bit1=HAS_AO, bit2=HAS_NORMAL, bit3=HAS_ORM
v_importedMech = (inst.renderFlags >> 3) & 15u;   // was 7u
```

### CPU bind (`mclib/mech3d.cpp`)
```cpp
// ORM-1: carry the ORM slot. HAS_ORM bit (6) added ONLY if ormValid.
desc.slot8TexHandle = ormValid ? (uint32_t)ormTextureHandle : 0u;
if (ormValid) desc.renderFlags |= 0x40u;   // bit 6 = HAS_ORM
```

### Uniform / unit
- New sampler unit 8 (`u_mechOrmTex`).
- Strength uniform `u_ormStrength` (default 0 or 1; 0 = byte-identical).

## Stock degrade
- Missing ORM .ktx2 → `ormValid = false` → bit clear → shader skips the sample.
- No change to stock mechs or to mechs that only have albedo/normal/AO.

## Relationship to existing PBR path
- The StandardLit path already has `u_pbrOrmTex` (units 2/4) for the two-layer triplanar materials.
- This proposal is a **per-mech imported ORM** that coexists; it does not replace the triplanar PBR materials.
- When both are present, the imported ORM can modulate the final ao/rough/metal or be ignored if StandardLit is driving the material (design choice left open).

## Suggested commit message (when applied)
```
feat(render): optional ORM sampler for imported mechs

Mirror the HAS_NORMAL path with HAS_ORM (renderFlags bit 6,
v_importedMech bit 3, unit 8). Missing map → bit clear → no-op.
Stock path unchanged.
```

## Files
- `shaders/mech.frag`
- `shaders/mech.vert`
- `mclib/mech3d.cpp` (bind + flag)
- This sketch

**Next action:** Apply the sketch, or combine with a MECH-NORMAL-PILOT once a normal map is available.
