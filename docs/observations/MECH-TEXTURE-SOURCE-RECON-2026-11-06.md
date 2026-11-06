# Mech Texture Source Recon — 2026-11-06

**Slice:** Locate source TGAs and existing cook path for mech albedo uplift.  
**Status:** Recon complete. No assets cooked.  
**Goal:** Identify a repeatable path for AI-upscale → BC7 KTX2 that degrades to stock.

## Existing cook tools (`tools/mc2texcook/`)

| Script | Purpose |
|--------|---------|
| `cook_tgl_tiers.py` | Batch cook TGAs from a zip or dir into `data/tgl/{256,512,1024}/*.ktx2` (BC7). Never upscales. |
| `cook_tgl_upscale_single.py` | Single-file Lanczos upscale + BC7 cook (or `--no-upscale`). |
| `batch_upscale_cook.py` / `batch_cook.py` | Broader batch helpers. |
| `cook_pbr_maps.py` | PBR map cook (normal/ORM). |

All require the KTX-Software `ktx` CLI (default path in scripts: `A:/Games/mc2-tools/ktx/ktx.exe`).

## Known good example

- Source: `data/tgl/128/fireantrgb.tga` (or extracted from release zips)
- Cooked: `data/tgl/128/fireantrgb.ktx2` (BC7, 512×512, AI-upscaled)
- Referenced in `docs/asset-pipeline.md` as the Flea GLB texture.

## Source locations (typical)

- Stock TGAs live in release asset zips (`mc2-tgl.zip` or equivalent) and/or `mc2srcdata/tgl/`.
- Deployed modern textures land in `data/tgl/{128,256,512,1024}/`.
- Loose-file override is the normal opt-in path; missing modern file falls back to stock TGA.

## Recommended pilot (next implementation slice)

1. Pick 2–3 stock mechs (Madcat, Vulture, Atlas).
2. Extract their primary albedo TGA from the release zip / `mc2srcdata`.
3. AI-upscale (or Lanczos for a baseline) to 512 or 1024.
4. Cook with `cook_tgl_upscale_single.py --no-upscale` (or tiers script) to BC7 KTX2.
5. Drop the `.ktx2` next to the expected load path.
6. Confirm stock fallback when the modern file is absent.

## Constraints

- Stock install must remain playable.
- Do not touch mech HUD icon atlases (`mcui_*`).
- Prefer BC7.
- All new textures are additive / loose-file optional.

## Files of record

- `tools/mc2texcook/`
- `docs/asset-pipeline.md`
- `docs/observations/HD-TEXTURE-RECON-2026-11-06.md`

**Next action:** Extract source TGAs for a pilot mech (requires the release asset zip or `mc2srcdata`) or pause for asset access.
