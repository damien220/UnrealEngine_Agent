# World Partition Grid & Data Layers

## World Partition Setup & Runtime Grid Configuration

Enable World Partition in `World Settings > World Partition > Enable World Partition` (checkbox). World Partition uses `UWorldPartition` class (found in `Engine/Source/Runtime/Engine/Public/WorldPartition/WorldPartition.h`) to manage spatial streaming and actor loading.

The runtime grid uses `FWorldPartitionRuntimeSpatialHash` to organize actors into cells. Grid cell size is configured in `World Settings > World Partition > Runtime Settings > Grids > Cell Size` — this value represents world units that make up each cell width/length. Community best practices recommend:

- **Urban/dense areas**: 32–64m cells (tight culling, more overhead from frequent loads)
- **Open terrain**: 128m cells (balanced streaming cost vs. draw distance)
- **Sparse/mountains**: 256m+ cells (fewer simultaneous loads, longer individual load times)

City Sample project demonstrates the recommended setup with a **256m × 256m × 256m** cell size and **768m loading range**. The rule of thumb: cell size should be a multiple of your Landscape component size (typically 128m component = 256m world partition cell). Too many small cells increase streaming overhead; too few large cells cause long hitches when crossing cell boundaries.

## Data Layers: Editor & Runtime Management

`UDataLayerAsset` defines a shareable data layer that can be painted onto actors in the level. `AWorldDataLayers` is the actor that manages data layer state at runtime. Editor vs. Runtime distinction:

- **Editor**: Data layers are organizational — groups of actors you can hide/show in the editor
- **Runtime**: Data layers are streamed independently via `UDataLayerSubsystem::SetDataLayerRuntimeState()`

Runtime states are enum `EDataLayerRuntimeState`:
- `Unloaded` — not loaded, not visible
- `Loaded` — in memory but hidden/inactive
- `Activated` — fully loaded and visible/active

Use data layers for: time-of-day variants (day/night lighting layouts), mission-specific layout changes, destruction state progression (before/after collapse), environmental toggles (weather states). Example: paint destructible buildings into a `Destroyed` data layer, gameplay code calls `UDataLayerSubsystem::SetDataLayerRuntimeState(DestroyedLayer, EDataLayerRuntimeState::Activated)` when triggered.

Data layers are replicated in multiplayer — all clients see the same state. Server authority: only the server calls `SetDataLayerRuntimeState`, changes replicate to all clients.

## Always-Loaded Layer & Gameplay-Critical Actors

Actors with `Is Spatially Loaded = False` in the Details panel are **always loaded** — they never stream out, regardless of player distance. Actors in a data layer designated as **Always Loaded** (set in the data layer asset properties) stream as a single permanent group.

Rule: **Gameplay-critical actors must be in Always Loaded**. Examples: player spawn points, essential audio managers, world lighting (sky, directional light), collision volumes for level boundaries. Leaving a required actor out of Always Loaded causes gameplay breaks when the cell unloads.

Anti-pattern: Landscape actors should **never** be in Always Loaded — they belong in regular data layers to stream with the world. Locking landscape Always Loaded wastes memory for invisible far-away terrain.

Actors referenced in the **Level Blueprint** are automatically marked Always Loaded (this is a footgun — use Blueprint Classes for dynamic actors instead, which do not auto-load).

## One File Per Actor (OFPA) Folder Structure

When World Partition is enabled, actors are saved as individual `.uasset` files under `Content/_ExternalActors_/MapName/`. Folder structure:
```
Content/
├── Maps/
│   └── MyLevel.umap          ← level metadata only
└── _ExternalActors_/
    └── MyLevel/
        ├── 1/                ← grid cells
        │   ├── A1B2C3D4E5F6G7H8I9J0K1L2M3.uasset
        │   └── N4O5P6Q7R8S9T0U1V2W3X4Y5Z6.uasset
        ├── 2/
        │   └── ...
        └── AlwaysLoaded/
            ├── SkyBox.uasset
            └── LevelLighting.uasset
```

Each actor file is encoded with a hash to avoid collisions. **Benefit**: two team members can edit different actors in the same cell concurrently without merge conflicts. **Source control**: track `_ExternalActors_/**` as binary assets; do not manually diff individual actor files.

Actors must be submitted to source control **from within the Editor** (drag-drop into source control panel or use the Editor's built-in integration). Perforce offers changelist validation — review actor names and levels before submit.

## Streaming Sources & Custom Streaming

By default, each `APlayerController` is a streaming source — the cells around the player are loaded. Enable `Player Controller > Enable Streaming Source` (default: true).

For custom streaming sources (e.g., AI listener, audio manager, teleport destination), add a `UWorldPartitionStreamingSourceComponent` to an actor. Set `SourceCellBehavior` to define loading radius in cells (1 cell = cell size, so 2 cells = 2× cell size loading radius).

Example: place an `AWorldPartitionStreamingSource` at a dungeon entrance — the code calls `StreamingSourceComponent->SetStreamingSourceActive(true)` to pre-load the dungeon before the player enters. This eliminates hitches.

## Debug Commands

- `wp.Runtime.ToggleDrawRuntimeHash2D` — overlay 2D grid cells in the viewport (top-down view). Green = loaded, red = unloaded. Essential for understanding streaming.
- `wp.Runtime.ToggleDrawRuntimeHash3D` — show 3D grid in perspective view.
- `wp.Runtime.ShowRuntimeSpatialHashGridLevel <N>` — highlight a specific grid level (0 = base, higher = LOD grids). Used with 3D grid to see multi-level hierarchy.
- `wp.Runtime.ForceGarbageCollection` — trigger garbage collection to test actor unload cleanup in PIE.
- `stat WP` — show World Partition streaming stats (cells loaded, pending cells, memory).

## Version Notes

- **UE5.0–5.1**: World Partition is core but 2D Runtime Hash only; 3D hash added in 5.2+.
- **UE5.2+**: Data Layers have full runtime state management; earlier versions had limited support.
- **UE5.3+**: OFPA folder structure stabilized, source control integration improved.

## Common Gotchas

1. **Slow cell loads**: Cell size too large (> 256m in dense areas) or too many actors per cell. Split into smaller cells or move non-critical actors to higher LOD data layers.
2. **Always-Loaded bloat**: Accidentally marking decorative actors Always Loaded bloats memory. Audit Always Loaded data layer regularly.
3. **Data layer references**: Gameplay code tries to find an actor in a data layer that's Unloaded. Check `IsValid()` before access; wait for `OnDataLayerStateChanged` delegate if dependent.
4. **Perforce merge conflicts**: Editing the same actor in two branches causes merge conflicts in the encoded `.uasset` file. Use branching strategies to isolate actor edits, or resolve via "ours" / "theirs" in Perforce conflict resolution.

