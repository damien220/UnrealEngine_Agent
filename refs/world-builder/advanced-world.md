# Large World Coordinates & OFPA in Production

## Large World Coordinates: When, Why & How

**Large World Coordinates (LWC)** enable worlds **> ~20km** in any axis. Without LWC (UE4 legacy), floating-point precision errors become visible at ~20km distance: camera jitter, physics vibration, vertex position errors. LWC uses **64-bit doubles** for world positions, eliminating precision loss.

**Enabled by default in UE5** — no project setting needed. In C++:

- **`FVector`** = double-precision (X, Y, Z as `double`)
- **`FVector3f`** = single-precision float (use for GPU-bound data: vertex positions, particle positions, skinned mesh bones)

**Never pass `FVector` directly to `FVector3f` parameters without explicit conversion**:

```cpp
// WRONG: loses precision
FVector WorldPos = {1000000.0, 2000000.0, 500000.0};
FVector3f GPUPos = FVector3f(WorldPos); // Compiles, but loses data silently

// CORRECT: explicit cast preserves intent
FVector3f GPUPos = FVector3f(WorldPos); // Use when intentional
// OR: use FLargeWorldCoordinates helper
FVector3f Relative = (WorldPos - CameraPosition).ToFloat(); // Camera-relative
```

### World Size Practical Limits

- **LWC disabled (UE4 legacy)**: Max ~21km without precision errors
- **LWC enabled**: Theoretical max 88 million kilometers, but practical limits:
  - **4km²** with full Lumen + Nanite at 60fps (typical shipped game)
  - **64km²** possible with streaming + HLOD + reduced Lumen range
  - **> 64km²**: Use origin rebasing or divide into separate levels per region (e.g., Artemis style)

Physics and rendering are most sensitive; streaming performance matters more than LWC coordinate size.

## Tile Origin Rebasing & Camera-Relative Rendering

**UWorld::OriginLocation** stores the current world tile origin. As the player moves far from origin, `AWorldSettings` periodically triggers **origin rebasing** to keep player position near zero (for physics precision).

Configure in **World Settings > World > Enable World Origin Rebasing** (checkbox) and **Initial World Origin Offset** (offset from world center on spawn). Rebasing happens automatically when player distance exceeds threshold.

Example flow:

```cpp
// Player at (10,000,000, 5,000,000, 0)
UWorld::OriginLocation = (10,000,000, 5,000,000, 0)
// Physics engine sees player at (0, 0, 0) locally

// Player moves further, rebase triggers
UWorld::OriginLocation = (11,000,000, 5,000,000, 0)
// All actor positions shifted -1,000,000 in X
// Physics still sees local near-zero position
```

For gameplay code, rebase is transparent — `GetActorLocation()` returns world space. Only GPU vertex positions and physics bodies care about local precision. **Never manually adjust `UWorld::OriginLocation`** — let the engine handle it.

### Niagara & Particle Rebasing

Niagara particles use float positions (not double). They store position as:
```glsl
Position = SystemPosition + (GridCell * CellSize)
```
Where `GridCell` is an int32 identifying which spatial cell the particle occupies. Rebasing updates `GridCell` automatically. No manual intervention needed.

## OFPA in Practice: Source Control & Workflow

**One File Per Actor** stores each actor in `_ExternalActors_/MapName/GridCellID/EncodedActorName.uasset`. Folder structure:

```
Content/
├── Maps/
│   └── MyOpenWorld.umap      ← level metadata only (small, < 5KB)
└── _ExternalActors_/
    └── MyOpenWorld/
        ├── 1/
        │   ├── A1B2C3D4E5F6.uasset  ← Actor encoded as hash
        │   ├── G7H8I9J0K1L2.uasset
        │   └── ...
        ├── 2/
        │   └── M3N4O5P6Q7R8.uasset
        └── AlwaysLoaded/
            ├── SkyBox_52C9.uasset
            └── LevelLights_A7F2.uasset
```

Each `.uasset` is typically 5–50KB depending on actor complexity. Advantage: **two team members editing different actors no longer merge conflict**. 

### Source Control Integration

**Perforce (recommended)**:
- Submit actors from **Editor > Source Control > Submit Content**
- Use **View Changelist** window to inspect actor names/types before submit (not the hashed filenames)
- Supports atomic multi-file submissions (all actors from a commit together)

**Git** (supported but less ideal):
- Track `_ExternalActors_/**` as binary files (prevents merge conflicts)
- `.gitignore` pattern: `_ExternalActors_/` → commit separately via `.lfs` or binary strategy
- Collaborative editing: use branching to isolate actor edits (different branches per team member's region)

**Best practice**: Commit `.umap` + related `_ExternalActors_` together in one changelist to maintain consistency.

### LWC + OFPA Interaction

OFPA file paths are relative to level grid cells (WorldPartition cell IDs). LWC affects:
- Actor positions are stored as doubles in the `.uasset`
- World origin rebasing shifts all actor positions transparently (source control unchanged)
- No explicit LWC handling needed in OFPA workflow

**Gotcha**: If manually rebasing origin (bad practice) before submitting, actor positions change on disk → confusing diffs. Never call `ApplyWorldOffset()` except in engineered scenarios (e.g., in-editor coordinate system conversion).

## Best Practices for 4km²+ Worlds

1. **Enable World Partition** (automatic LWC, OFPA) — never build large worlds without it.
2. **Test with `wp.Runtime.ToggleDrawRuntimeHash2D`** before shipping — verify cell streaming works.
3. **Pin cells in PIE** for deterministic testing, unpin before final ship verification.
4. **Monitor physics load** — use `r.Physics.DefaultWorldGravity` and collider complexity per region (complex collision in sparse mountains, simple in dense areas).
5. **Split landscapes if > 4km²** — one landscape per 2km² region (separate Landscape actors, linked by World Partition cells). Each landscape has own LOD budget.
6. **Use LWC intentionally** — if world < 20km and no precision errors, explicit `FVector3f` conversions are optional. Above 20km, require them.
7. **Profile memory per cell** — `stat Streaming` to confirm unload cleanup. Leaking cells bloat memory over time.

## Version Notes

- **UE5.0**: LWC enabled, OFPA enabled by default in World Partition. Origin rebasing automatic.
- **UE5.1–5.2**: LWC stability improved, Niagara rebasing finalized.
- **UE5.3+**: Origin rebasing threshold tuning available, multi-level World Partition grids with LWC.

## Common Gotchas

1. **FVector → FVector3f precision loss**: Cast explicitly. Use `(WorldPos - CameraPos).ToFloat()` to maintain relative precision.
2. **Physics breaking at extreme distances**: If rebase disabled or threshold too high, physics jitter visible at 20km+. Enable rebasing.
3. **OFPA merge conflicts with Perforce**: Two edits to same actor in different branches cause binary conflict. Use P4 smart merges or resubmit one edit on top of the other (linear history).
4. **Landscape > 1 actor**: Splitting landscape across multiple actors with different cell origins causes seam artifacts. Use single Landscape per region, accept memory cost.
5. **Multiplayer position sync**: With rebasing, `GetActorLocation()` world space is consistent. Network replication handles it automatically. Only problem if custom code caches world positions across rebases (don't do that).

