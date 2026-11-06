# HD Texture Pipeline Recon — 2026-11-06

**Slice:** Texture/HD inventory for 2026-standards remaster.  
**Status:** Recon complete. No code changes.  
**Goal:** Identify remaining low-res surfaces and a safe, stock-degrading path to higher fidelity.

## Current state (from docs/asset-pipeline.md, verified 2026-06-17)

| Surface | Current res | Upscaled? | Notes / constraints |
|---------|-------------|-----------|---------------------|
| Terrain colormaps / burn-ins | 5120² atlas (4× from 1280²) | Yes (ESRGAN) | BC7, GPU indirect path. Solid. |
| Terrain detail / normals / overlays | 64²→256² | Yes | Overlay tiles have gated 256² path (`MC2_OVERLAY_TILE_HIRES`). |
| Static prop / building textures | Tiered (128–1024) | Partial | BC7 cook path exists (`tools/mc2texcook`). |
| Mech textures | Mixed; some 4× archives, Flea example at 512² BC7 | Partial | AI-upscaled examples exist. Most still stock or low-tier. |
| Mech HUD icon atlases (`mcui_med4` etc.) | Stock cell sizes | **No** | Explicitly not upscalable — direct-indexed sprite sheets. Non-uniform pitch crashes. |
| UI / pilot portraits / force-group | Stock | No / partial | Must preserve atlas layout and stock fallback. |
| Water / masks | 64²→256² | Yes | |

## Highest-impact remaining surfaces for “HD graphics”

1. **Mech albedo / normal / roughness sets**  
   - Biggest visual win at zoom levels players actually use.  
   - Existing cook tools (`tools/mc2texcook/cook_tgl_tiers.py`) already support multi-tier BC7.  
   - Pattern: AI upscale source TGAs → cook to 512/1024 BC7 KTX2 → opt-in via existing `[Import]` or texture override path.  
   - Must degrade: missing modern texture → stock TGA path.

2. **Static prop / building texture uplift** (where not already tiered)  
   - Same cook pipeline. Lower priority than mechs because they are more numerous but often viewed at distance.

3. **Lighting / shader polish** (not pure resolution)  
   - Existing PBR splatting, tessellation, CSM shadows, reverse-Z.  
   - Next: consistent material library, better normal blending on mechs, optional SSR/SSAO if frame budget allows.  
   - Owned by shader / render experts; not blocked by texture cook.

4. **UI / icons**  
   - Low priority for first HD pass.  
   - Mech icons and many HUD atlases are layout-sensitive.  
   - Separate “UI scale / resolution independence” slice (later).

## Recommended next slices (autonomous-friendly)

1. **MECH-TEXTURE-UPSCALE-PILOT**  
   - Pick 2–3 stock mechs (e.g. Madcat, Vulture, Atlas).  
   - Document source TGA locations in `mc2srcdata` / release zips.  
   - Produce a repeatable upscale + `cook_tgl_tiers` recipe that outputs opt-in BC7 sidecars.  
   - Add a short validation note: visual parity at stock zoom, improvement at close zoom, stock fallback when modern file missing.

2. **MATERIAL-LIB-AUDIT**  
   - Inventory which mechs already have normal/roughness maps vs pure diffuse.  
   - Propose a minimal material schema extension that the existing PBR path can consume without breaking stock.

3. **STOCK-DEGRADE GATE**  
   - Confirm every new modern texture path has a missing-file fallback that keeps the stock install playable (north-star constraint).

## Constraints (do not violate)

- Stock install must remain playable. Modern assets are additive/opt-in.
- Do not upscale mech HUD icon atlases (`mcui_*`) — they are direct-indexed.
- Prefer BC7 for GPU-native formats; ETC1S only where already policy (burn-ins).
- All new paths must be killswitchable or loose-file optional.

## Files / tools of record

- `docs/asset-pipeline.md` (canonical inventory)
- `docs/asset-modernization-pipeline.md`
- `tools/mc2texcook/`
- `tools/overlay_upscale/`
- `mclib/terrtxm2.cpp`, `mech3d.cpp`, `txmmgr.cpp`

**Next action:** Begin MECH-TEXTURE-UPSCALE-PILOT recon (source locations + cook recipe) or move to Brain/AI tactic review.
