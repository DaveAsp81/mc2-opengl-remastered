# Material Library Audit — 2026-11-06

**Slice:** Inventory current PBR / normal / ORM support vs pure-diffuse mechs.  
**Status:** Audit complete. No assets added.  
**Goal:** Identify the smallest material schema extension that the existing PBR path can consume without breaking stock.

## Current state

### Mech rendering (`shaders/mech.frag`)
- Albedo: `u_tex` (existing paint-hash / TGA / KTX2 path).
- AO: `u_mechAoTex` (unit 6) + `u_aoStrength`. Active only when the CPU sets the HAS_AO bit.
- Normal: `u_mechNormalTex` (unit 7) + `u_normalStrength`. Active only when HAS_NORMAL bit is set (i.e. the .ktx2 loaded). Missing artifact → geometric normal fallback.
- Specular uniforms exist but are inert at strength 0 in the non-viewuniforms path.
- **No dedicated ORM sampler** visible in the current mech.frag fragment.

### Material library
- `gameassets/materials/CorrugatedSteel006A/` — full PBR pack (albedo_srgb, normal_linear, orm_linear) cooked via `tools/mc2texcook/cook_pbr_maps.py`.
- `data/materials/pbr/` contains the corresponding KTX2 files.
- Currently oriented toward buildings / terrain; not yet wired as a per-mech material override.

### Mech textures today
- Most mechs remain pure diffuse (paint-hash TGAs or tiered KTX2 albedos).
- Flea example has an AI-upscaled albedo (`fireantrgb.ktx2`) but no accompanying normal/ORM in the inventory.
- 4× archives exist; BC7 cook path is mature.

## Gaps

1. **Normals on mechs** — shader support exists, but almost no mech has a normal map deployed.
2. **ORM / roughness** — cook tool exists (`cook_pbr_maps.py`), but mech.frag has no ORM sampler yet.
3. **Material assignment** — no per-mech or per-slot material override that points at a library entry. Current path is still “one albedo texture per paint scheme.”
4. **Stock degrade** — any new map must be optional; missing file → bit clear → geometric normal / no AO.

## Minimal next slices (autonomous-friendly)

1. **MECH-NORMAL-PILOT**  
   - Generate or source a normal map for 1–2 mechs (Madcat, Flea).  
   - Cook to BC7 linear KTX2.  
   - Confirm the existing HAS_NORMAL path picks it up and falls back cleanly.

2. **ORM-SAMPLER-RECON**  
   - Decide whether to add a single `u_mechOrmTex` (R=AO, G=roughness, B=metal) to mech.frag, or keep AO separate and add only roughness.  
   - Sketch the CPU-side bit + uniform bind (mirror the existing normal path).

3. **MATERIAL-OVERRIDE-HOOK**  
   - Small ini / sidecar field (`Material=CorrugatedSteel006A` or per-slot) that the mech loader can honor.  
   - Default empty → current albedo-only behavior.

## Constraints

- Stock install must remain playable.
- Missing modern maps must degrade (bit clear, geometric normal, no AO).
- Prefer reusing the existing cook pipeline and the HAS_NORMAL / HAS_AO pattern.
- Do not require new maps for stock mechs.

## Files of record

- `shaders/mech.frag` / `shaders/mech.vert`
- `tools/mc2texcook/cook_pbr_maps.py`
- `gameassets/materials/CorrugatedSteel006A/`
- `docs/asset-pipeline.md`
- `docs/observations/HD-TEXTURE-RECON-2026-11-06.md`
- `docs/observations/MECH-TEXTURE-SOURCE-RECON-2026-11-06.md`

**Next action:** MECH-NORMAL-PILOT (source or generate one normal) or ORM-SAMPLER-RECON (shader + bind sketch).
