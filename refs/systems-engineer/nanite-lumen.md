# Nanite & Lumen — UE5 Reference

## Nanite Overview

Nanite is UE5's virtualized geometry system. It renders only the triangles visible at current pixel resolution, eliminating manual LOD authoring for static geometry. It is not a universal geometry solution — it has hard constraints that must be planned around.

---

## Nanite Hard Limits

### Instance Budget — 16 Million
Nanite supports a hard-locked maximum of **16 million instances** per scene. This is a GPU-side streaming limit, not a recommendation.

| Scene Type | Typical Risk |
|---|---|
| Corridor shooter | Low — limited geometry variety and density |
| Urban open world | Medium — modular kits multiply fast |
| Dense foliage open world | **High** — grass + rocks + trees across a 4km² cell hits budget |

**Planning rule**: Count instances per World Partition cell, not per level. A 4km² map streaming 3 active cells simultaneously triples your effective instance count.

### Tangent Space
Nanite derives tangent space in the pixel shader from screen-space derivatives. Do **not** store explicit tangents on Nanite meshes — the data is unused and increases memory.

In the Static Mesh Editor: `Build Settings → Recompute Tangents` is irrelevant for Nanite-enabled meshes. The engine ignores stored tangents at runtime.

---

## Nanite Incompatibilities

These asset or material types are **not compatible** with Nanite and require standard LODs:

| Type | Why Incompatible | Fallback |
|---|---|---|
| Skeletal meshes | Deformation requires per-vertex transform — Nanite is screen-space only | Standard LOD + skin cache |
| Spline meshes | Procedural deformation is applied post-rasterization | Baked static mesh segments |
| Procedural Mesh Component | Runtime-generated geometry bypasses Nanite streaming | Convert to `UStaticMeshComponent` at bake time |
| Masked materials with complex clip | `clip()` / `discard` requires full triangle to be rasterized; Nanite's culling assumes opaque | Benchmark: disable Nanite if overdraw cost > standard LOD cost |
| Translucent materials | Nanite is opaque-pass only | Separate translucent mesh, no Nanite |
| Two-sided foliage with subsurface | Subsurface scattering pass is incompatible with Nanite's deferred resolve | Use standard foliage LOD pipeline |

**Masked material guidance**: Enable Nanite on masked materials only if the geometry has high polygon complexity AND the clip pattern is simple (e.g., billboard card leaves). Always A/B profile with `stat Nanite` before committing.

---

## Nanite Compatibility Validation

Validate Nanite eligibility before content lock:

```cpp
// Editor-only utility — runs during asset validation pipeline
#if WITH_EDITOR
void UMyAssetValidator::ValidateNaniteCompatibility(UStaticMesh* Mesh)
{
    if (!IsValid(Mesh)) return;

    const FMeshNaniteSettings& NaniteSettings = Mesh->GetNaniteSettings();

    if (!NaniteSettings.bEnabled)
    {
        UE_LOG(LogMyGame, Warning,
            TEXT("Mesh '%s' is not Nanite-enabled. Verify LODs are configured."),
            *Mesh->GetName());
        return;
    }

    // Warn if ray tracing is enabled without Nanite fallback mesh
    if (Mesh->bSupportRayTracing && !Mesh->HasValidRenderData())
    {
        UE_LOG(LogMyGame, Warning,
            TEXT("Mesh '%s': Ray tracing enabled but render data missing — check Nanite fallback."),
            *Mesh->GetName());
    }

    UE_LOG(LogMyGame, Log,
        TEXT("Nanite scene budget: 16M instance limit. Mesh '%s' validated."),
        *Mesh->GetName());
}
#endif
```

**Enable early in production**: Run `r.Nanite.Visualize overview` in every level as a weekly check from first content drop. Catching an incompatible material at week 2 costs one sprint. Catching it at content lock costs three.

---

## Nanite Visualization & Profiling Commands

```
r.Nanite.Visualize overview          — shows instance count, streaming state, culled geometry
r.Nanite.Visualize triangles         — heatmap of triangle density on screen
r.Nanite.Visualize primitives        — per-primitive Nanite status
r.Nanite.Visualize clusters          — cluster-level culling visualization

stat Nanite                          — frame counters: instances drawn, triangles, streaming I/O
stat NaniteScene                     — per-scene aggregate stats
```

**Profiling workflow**: Run `stat Nanite` before and after each major content addition. Track `Instances Drawn` as your primary budget counter. If it approaches 12M (75% of limit), audit foliage density and instance-heavy modular kits.

---

## Where Nanite Excels

Use Nanite aggressively for:
- **Dense static foliage** — rocks, gravel, ground clutter, fallen logs (not animated vegetation)
- **Modular architecture kits** — walls, floors, props, facades with high polygon counts
- **Terrain detail meshes** — cliff faces, rock formations, rubble
- **Hero props** — high-detail static hero objects (vehicles, machines) without animation

Avoid Nanite for anything that moves, deforms, clips, or is translucent. Those asset classes have purpose-built pipelines (skin cache, spline mesh baking, masked LOD cards, translucency sorting).

---

## Lumen Overview

Lumen is UE5's fully dynamic global illumination and reflections system. It requires no lightmap baking and responds to dynamic light changes in real time. It runs on the game thread's rendering path and has its own scalability settings.

---

## Lumen Configuration

### Enable in Project Settings
```ini
; DefaultEngine.ini
[/Script/Engine.RendererSettings]
r.DynamicGlobalIlluminationMethod=1    ; 0=None, 1=Lumen, 2=SSGI
r.ReflectionMethod=1                   ; 0=None, 1=Lumen, 2=SSR
r.Lumen.HardwareRayTracing=1           ; Enable if target hardware supports DXR
```

### Per-PostProcessVolume Settings
```
Lumen Global Illumination → Method: Lumen
Lumen Reflections → Method: Lumen
Lumen Scene Detail → controls micro-detail lighting (default 1.0, raise for interiors)
Lumen Scene View Distance → max GI propagation distance (default 20000 cm = 200m)
Final Gather Quality → 1.0 standard, raise for cinematic, lower for performance
```

### Lumen with Sky Light
Lumen requires a Sky Light in the scene to correctly propagate outdoor indirect lighting. Without it, open-sky scenes will have flat, incorrect GI.

```
Sky Light → Source Type: SLS_CapturedScene
Sky Light → Real Time Capture: Enabled (for dynamic sky/weather)
```

---

## Lumen Incompatibilities & Caveats

| Issue | Cause | Resolution |
|---|---|---|
| Interior dark spots | Missing or occluded Lumen scene probes | Add `r.Lumen.SceneCaptureCacheResolution` increase; add fill lights |
| Lumen doesn't update at distance | View distance cap | Increase `Lumen Scene View Distance` in PostProcessVolume |
| Lumen + Nanite: flickering on masked geo | Masked materials in Lumen hit path | Switch masked foliage to standard LOD or raise `r.Lumen.TraceMeshSDFs` threshold |
| Performance: Lumen SW ray tracing too slow | Software fallback on non-DXR hardware | Enable `r.Lumen.HardwareRayTracing=1` on supported hardware; drop Final Gather Quality on low-end |
| Reflections missing on dynamic actors | Screen-space fallback for fast-moving objects | Expected behavior; use SSR as fallback for specific objects |

---

## Lumen Profiling Commands

```
stat Lumen                           — frame budget breakdown for Lumen GI and reflections
r.Lumen.Visualize.Overview 1         — shows Lumen probe placement and coverage
r.Lumen.Visualize.RadianceCache 1    — radiance cache debug view
r.Lumen.DiffuseIndirect.Allow 0      — toggle GI off for A/B profiling
r.Lumen.Reflections.Allow 0          — toggle reflections off for A/B profiling
```

**Optimization target**: Lumen + Nanite combined should consume ≤ 4ms GPU on target hardware at 1080p. Profile both independently before combining them in a scene.
