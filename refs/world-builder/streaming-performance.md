# Streaming Performance Profiling & Optimization

## World Partition Streaming Diagnostics

Essential profiling tools:

1. **`wp.Runtime.ToggleDrawRuntimeHash2D`** — Overlay 2D grid cells (top-down in editor viewport). Green = loaded, red = unloaded, yellow = loading. Essential for understanding streaming patterns.

2. **`wp.Runtime.ToggleDrawRuntimeHash3D`** — 3D grid overlay in perspective view (shows z-axis streaming). Use with multi-level streaming hierarchies (UE5.2+).

3. **`wp.Runtime.ShowRuntimeSpatialHashGridLevel <N>`** — Highlight specific grid LOD level. 0 = base cells, 1 = larger cells (if multi-level streaming configured). Helps understand cell hierarchy.

4. **`stat WP`** — Console stat showing:
   - Cells loaded / total cells
   - Cells loading (pending)
   - Memory used by streaming
   - Streaming source count

5. **`stat Streaming`** — Asset-level streaming stats (texture pools, load requests, async loads in flight).

6. **Unreal Insights** — Advanced profiling:
   - Enable `AssetLoading` trace channel
   - Search `Game Thread` track for streaming hitches (spikes in frame time)
   - Identify which actors/assets cause stalls
   - Compare frame time before/after cell load

## Streaming Hitches: Causes & Fixes

### Hitch Pattern 1: Too-Small Cells

**Symptom**: Frequent brief hitches (~50–200ms) as player walks, multiple cells loading simultaneously.

**Root cause**: Cell size too small (< 64m in dense areas). More cells loaded concurrently = more parallel async loads = thread pool saturation.

**Fix**: Increase cell size to 128m–256m. Fewer simultaneous loads. Trade-off: unload distance increases (less aggressive unloading of far cells).

### Hitch Pattern 2: Too-Large Cells

**Symptom**: Infrequent but severe hitches (~500ms–2s) when crossing cell boundary. One giant cell loads.

**Root cause**: Cell size too large (> 256m in dense areas, > 1km in sparse). Single cell contains 1000+ actors. Takes long to deserialize + spawn.

**Fix**: Reduce cell size, or split dense content into separate data layers (streaming separately). Pre-load adjacent cells using `UWorldPartitionStreamingSource` before camera enters.

### Hitch Pattern 3: Synchronous Asset Loads

**Symptom**: Stall when entering cell, async load appears to block.

**Root cause**: Gameplay code tries to access actor/asset in a loading cell, forcing synchronous completion of async load. Example: `TryGetSoftPtr()` on an unloaded level.

**Fix**: Defer gameplay logic until cell finishes loading. Listen to `AWorldPartition::OnCellStateChanged` delegate, wait for all required data layers to reach `EDataLayerRuntimeState::Activated`.

## Streaming Load Throttling

Configure async loading concurrency via console variables:

- **`r.LevelStreaming.MaxRequestsPerFrame`** — Max async load requests queued per frame (default: 5). Lower = less stutter but slower loads. Higher = more parallel loads but potential hitches.
- **`r.LevelStreaming.LoadingPriorityFactor`** — Weight for priority-based loading (0.0–1.0). Higher = respect stream priority more, load nearby cells first.
- **`s.MaxPackagesToLoad`** — Global cap on async package loads in flight (default: 8). Limits thread pool saturation across entire engine.

Example tuning for 60fps target with 128m cells:

```
r.LevelStreaming.MaxRequestsPerFrame=3       // Load 3 cells max per frame
r.LevelStreaming.LoadingPriorityFactor=0.8   // Prioritize nearby
s.MaxPackagesToLoad=6                        // Cap global loads
```

Monitor `stat Streaming` to ensure "Active Loads" stays < `MaxPackagesToLoad`.

## Texture Streaming Pool & Exhaustion

Texture streaming pool holds textures for loaded levels. Configure via:

- **`r.Streaming.PoolSize`** — Max texture memory in MB (default: 1024MB on console, 2048MB on high-end PC). Set via `DefaultEngine.ini`:
  ```ini
  [/Script/Engine.Engine]
  r.Streaming.PoolSize=4096
  ```

**Exhaustion symptoms**:
- Textures appear blurry/low-res
- `stat Streaming` shows `StreamingPool: Used > Max`
- Hitch when camera pans (stalling to load high-res textures)

**Fix**:
1. Increase pool size if VRAM available
2. Use `r.Streaming.FullyLoadUsedTextures=1` to prioritize visible textures (can cause longer hitches)
3. Reduce texture sizes in content (1k instead of 4k for distance props)
4. Use World Partition to unload distant level textures

Monitor with `stat Streaming` continuously — pool exhaustion is cumulative across all loaded cells.

## Unreal Insights Streaming Analysis

1. **Start trace**: `Trace.Start -File=/Logs/MyStreamingTrace`
2. **In-game profiling**: Play level, move camera to trigger streaming events
3. **Stop trace**: `Trace.Stop` or `Trace.Pause`
4. **Analyze**: Open `/Logs/MyStreamingTrace.utrace` in Unreal Insights
5. **Look for**:
   - `AssetLoadTime` track: vertical bars = load events. Long bars = slow loads.
   - `Game Thread` track: spikes correlating with asset loads = blocking loads (bad).
   - Filter by actor name to trace specific actor's load time.

## Editor Testing & PIE Optimization

- **Pin cells for determinism**: World Partition Editor UI → right-click cells → Pin (forces load in PIE). Ensures consistent testing.
- **`wp.Runtime.ForceGarbageCollection`** — Trigger GC in PIE to test unload cleanup. Verify no memory leaks after cell unload.
- **Preview packaging**: Package small map region (`File > Package Project > Map`), measure load times on target device.

## Cell Size Recommendations Table

| Terrain Type | Cell Size | Loading Range | Notes |
|---|---|---|---|
| Dense urban | 32–64m | 192–256m | Tight culling, frequent loads |
| Open fields | 128–256m | 384–512m | Balanced, typical open world |
| Mountains/sparse | 256–512m | 512–1024m | Few large loads, longer hitches |
| Dungeons | 32m | 96m | Tight control, fast transitions |

## Version Notes

- **UE5.0–5.1**: Basic World Partition streaming; cell size tuning essential for performance.
- **UE5.2+**: Multi-level streaming grids, improved async scheduling, Unreal Insights `AssetLoading` trace.
- **UE5.3+**: Streaming pool auto-tuning, better texture priority scheduling.

## Common Gotchas

1. **Pool size mismatch**: Set `r.Streaming.PoolSize` too low for your cell size → constant texture thrashing. Rule of thumb: pool size ≥ 2× largest loaded level texture footprint.
2. **PIE cell pinning forgotten**: Test level with cells pinned, ship with unpinned → different performance characteristics. Always test with pin disabled before shipping.
3. **Synchronous loads on gameplay**: Quest code does `GetActorOfClass()` before actor's cell loads. Add `IsValid()` checks and defer gameplay until layer activation.
4. **Cascading unloads**: Unloading cell A triggers load of cell B (if B's actor references A). Check world graph for unintended data dependencies.
5. **Network latency**: Multiplayer + streaming can cause actors to stream in on some clients before others see them spawned on server. Ensure replication updates match streaming state.

