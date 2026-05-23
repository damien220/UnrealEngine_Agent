# HLOD Layers, Build Workflow & Nanite Integration

## HLOD Layers & Build Workflow

**HLOD Layer** (`AHLODLayer` actor or data layer type) defines LOD reduction strategy for a region. Configure via:

1. **World Settings > HLOD > HLOD Layers** — list of layers with settings:
   - **Min Screen Size** — When actor's screen size falls below this (0.1–1.0 typical), HLOD candidate
   - **Merge Method** — `Mesh Merge`, `Simplygon` (poly reduction), `Proxy LOD`, `Instancing` (for identical meshes)
   - **Generate Billboard** — Creates billboards for > 500m distance (imposter quads with pre-rendered images)

2. **Parent/Child HLOD Layers** — Chain LODs sequentially (e.g., Medium Detail → Far Detail → Billboard). Child layers reduce further, each with its own distance threshold.

**Build workflow**:
- Editor: `World > Build HLOD` (generates meshes under `_ExternalActors_/MapName/`)
- CLI: `RunUAT.bat BuildHLOD -Project=MyProject.uproject` (CI/CD)
- Timing: Rebuild after geometry changes in region. HLOD stale = pop-in visible on camera pan.

**Mesh Merge** combines many small actors (trees, rocks, props) into one draw call at distance. **Instancing** is cheaper for identical meshes — if 100 trees are the same asset, one instanced mesh < 100 merged meshes.

**Simplygon/ProxyLOD** reduces polygon count (e.g., 100k tri rock → 5k tri simplified rock). Use for complex static meshes; less useful for simple shapes that already merge well.

## HLOD vs. Nanite: Complementary, Not Competing

**Nanite** (per-mesh LOD) reduces triangles per individual mesh with automatic LOD hierarchy and hardware rasterization optimization. Achieves "1 pixel = 1 triangle" at any distance.

**HLOD** (batching LOD) merges multiple actors into one draw call and optionally reduces poly count further. Addresses two problems Nanite alone cannot solve:

1. **Draw call batching**: Nanite still has one draw call per unique mesh. HLOD merges 50 trees (50 Nanite draw calls) into 1 merged HLOD draw call.
2. **Extreme distance LOD**: Nanite's LOD hierarchy works well to 500m+, but HLOD billboards (simple quads) are cheaper beyond 500m+ (distant mountain silhouettes).

**Strategy**: Use **both**. Nanite per-mesh LOD for medium distance (10m–500m), HLOD for draw-call reduction and extreme distance (500m+). Nanite mesh with billboard HLOD is ideal.

## Imposter HLOD & Billboard Setup

Billboard imposters render **12 images around the UP axis** (360° coverage in 30° steps), captured with diffuse + normals baked in. At runtime, blend between nearest two views based on camera angle. Performance: 2 triangles + 1 texture lookup << complex mesh LOD.

Create imposter HLOD: Set HLOD Layer `Merge Method = Billboard`, configure **Imposter Atlas** in HLOD settings:

- **Atlas Size**: 2048×2048 typical (can go to 4096 for higher quality)
- **Tile Size**: 256×256 (imposter quad size in pixels; larger = more detail, more memory)
- **Image Count**: 12 (standard; more = higher quality but 12× memory per axis)

HLOD auto-generates imposter atlas on build. Inspect with `r.HLOD.DebugForceLevel <N>` (0 = original, 1 = first LOD, 2 = billboard).

## Debugging & Profiling

- **`r.HLOD.DebugForceLevel <N>`** — Force camera to render HLOD level N (0 = original, 1+ = LODs). Essential for testing transitions.
- **`stat HLOD`** — Shows HLOD stats: clusters created, memory used, merge method breakdown.
- **Nanite visualization**: `r.Nanite.Visualization.Mode` with options:
  - `TriangleCount` — Shows triangle count overlay (blue = few, red = many)
  - `Lod` — Shows LOD level (green = near LOD, red = far LOD)
  - `MaterialComplexity` — Per-pixel cost (impacts per-meshlet shading)

## HLOD Rebuild Triggers & CI Integration

- **Full rebuild**: After moving actors, changing materials, or adjusting HLOD Layer settings. Use `World > Build HLOD` in editor.
- **Incremental**: When only small regions change, manually select HLOD clusters and rebuild locally (not available in CLI).
- **CI/CD**: `RunUAT.bat BuildHLOD -Project=Project.uproject -Map=MapName -SkipSharedBuilds` rebuilds all maps. Typical build time: 10–30 minutes per 4km² region depending on complexity.

Monitor with `stat HLODx` (x = HLOD level) in packaged game to verify HLOD transitions are working.

## Version Notes

- **UE5.0–5.1**: HLOD Layers are available but UI/workflow less polished; Nanite integration basic.
- **UE5.2+**: HLOD Layer parenting (multi-level chains) fully supported; Nanite visualization tools improved.
- **UE5.3+**: Automatic grid-based HLOD generation (Roadmap feature); custom HLOD control advanced.
- **UE5.4+**: HLOD performance optimizations; faster build times, better imposter quality.

## Common Gotchas

1. **HLOD stale**: Rebuilding geometry but forgetting to rebuild HLOD → pop-in visible when zooming out. CI should fail-loud if HLOD rebuild time exceeds threshold.
2. **Overmerging**: Setting too-high merge threshold (e.g., 1000 small rocks into one HLOD) creates massive drawcall when HLOD loads. Break into regional HLODs instead.
3. **Imposter orientation**: Billboard imposters only work well for objects with clear UP axis (trees, pillars). Side-lying structures (bridges, walls) look wrong. Use poly-reduced HLOD instead.
4. **Nanite + HLOD conflict**: Applying HLOD to Nanite meshes whose LOD is already heavy-optimized may add little value. Profile with `stat HLOD` and `stat Nanite` to justify complexity.
5. **Billboard artifacts**: Imposter quads appear at shallow angles (camera perpendicular to billboard plane). Increase tile size or image count to reduce visible transitions.

