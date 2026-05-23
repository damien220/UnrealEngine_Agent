# Water System — Reference

## Overview

UE5's Water plugin provides `AWaterBody` actors (Ocean, Lake, River, Custom), procedural Water mesh generation, a physically-based Water material, and `UBuoyancyComponent` for floating actors. The plugin shipped as **Experimental** in UE5.0–5.2 and became stable in **UE5.3+**. Enable via `.uproject` — it is not on by default.

---

## Plugin Activation

```json
// YourGame.uproject
{
    "Plugins": [
        { "Name": "Water", "Enabled": true }
    ]
}
```

After enabling, add the module to `.Build.cs` if accessing Water C++ APIs:

```csharp
PublicDependencyModuleNames.AddRange(new string[] {
    "Water"   // AWaterBody, UBuoyancyComponent
});
```

---

## WaterBody Actor Types

| Actor | Use | Notes |
|---|---|---|
| `AWaterBodyOcean` | Infinite tiled ocean | **Must be in Always Loaded data layer** — streaming it causes mesh generation failures |
| `AWaterBodyLake` | Finite enclosed lake | Can be streamed with its cell |
| `AWaterBodyRiver` | Spline-driven river | Width and depth driven by spline tangents |
| `AWaterBodyCustom` | Arbitrary non-tiled surface | No auto-mesh generation; requires manual material plane |

### Always Loaded requirement for Ocean

```markdown
In World Partition Editor:
1. Create a Data Layer named "AlwaysLoaded" (Type: Always Loaded)
2. Select AWaterBodyOcean in the world
3. Assign it to the AlwaysLoaded data layer
4. Do NOT place it in any streaming cell

Rationale: AWaterBodyOcean drives tile mesh generation across the entire world.
If it streams out, all water tiles stop updating and the water surface disappears.
```

---

## Water Material Architecture

The Water plugin provides a master material `M_Ocean` (Engine content). For custom water:

1. **WaterInfoTexture** — auto-generated texture atlas encoding depth, flow vectors, and wave data. Sample it in your material to drive wave normals and depth-based color blending.
2. **SingleLayerWater** shading model — use `Single Layer Water` material domain for correct underwater depth fading and refraction at low cost.
3. **WaterBodyComponent** sets `WaterInfoTexture` at runtime. Never assign a static texture — let the plugin manage it.

```
Material domain: Single Layer Water
Shading model: Single Layer Water
Inputs:
  Base Color: Lerp(deep_color, shallow_color, depth_fade)
  Metallic: 0
  Roughness: 0 (fully specular)
  Normal: WaterWaveNormal (from WaterInfoTexture + tiling detail normals)
  Opacity: WaterVisibilityDepth (from depth buffer)
```

---

## UBuoyancyComponent

Provides pontoon-based buoyancy for boats, crates, and debris. Attach to any Actor.

```cpp
// MyBoat.h
#include "BuoyancyComponent.h"

UCLASS()
class MYGAME_API AMyBoat : public APawn
{
    GENERATED_BODY()

public:
    AMyBoat();

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Buoyancy")
    TObjectPtr<UBuoyancyComponent> BuoyancyComponent;

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Physics")
    TObjectPtr<UStaticMeshComponent> BoatMesh;
};

// MyBoat.cpp
AMyBoat::AMyBoat()
{
    BoatMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("BoatMesh"));
    RootComponent = BoatMesh;
    BoatMesh->SetSimulatePhysics(true);

    BuoyancyComponent = CreateDefaultSubobject<UBuoyancyComponent>(TEXT("Buoyancy"));
}
```

### Pontoon configuration (in editor or C++)

```cpp
// Configure pontoons at corners of hull — 4 pontoons for a small boat
// More pontoons = more stable but higher CPU cost
// Rule: minimum 3 pontoons (triangle), typical boat = 4–6

FSphericalPontoon FrontLeft;
FrontLeft.RelativeLocation = FVector(200.f, -80.f, 0.f);  // Relative to root
FrontLeft.Radius = 40.f;

FSphericalPontoon FrontRight;
FrontRight.RelativeLocation = FVector(200.f, 80.f, 0.f);
FrontRight.Radius = 40.f;

FSphericalPontoon RearLeft;
RearLeft.RelativeLocation = FVector(-200.f, -80.f, 0.f);
RearLeft.Radius = 40.f;

FSphericalPontoon RearRight;
RearRight.RelativeLocation = FVector(-200.f, 80.f, 0.f);
RearRight.Radius = 40.f;

BuoyancyComponent->BuoyancyData.Pontoons = { FrontLeft, FrontRight, RearLeft, RearRight };
```

**Pontoon count vs performance:**

| Pontoons | Stability | CPU cost per tick |
|---|---|---|
| 3 | Minimum stable | Lowest |
| 4–6 | Typical boat | Low |
| 8–12 | Large ship, detailed rocking | Medium |
| 16+ | Diminishing returns | High — avoid |

---

## Water Mesh LOD Configuration

Ocean tile generation is controlled via CVars:

```
r.Water.WaterMesh.ShowTileBounds 1           -- Debug: draw tile boundaries in viewport
r.Water.WaterMesh.NumSectionsInXY 2          -- Sections per tile in X and Y (default: 2)
r.Water.WaterMesh.LODCountBias 0             -- Add/subtract LOD levels globally (negative = more detail)
r.Water.WaterMesh.TessellationFactor 4       -- Tessellation subdivisions per tile (1–8, higher = smoother)
r.Water.WaterMesh.MaxWorldLODStrips 128      -- Caps total LOD strips across all tiles
```

**Performance guidelines:**
- Set `TessellationFactor` to 2–4 for mid-range targets; 4–8 only for hero shots
- `NumSectionsInXY=1` on mobile/console; `NumSectionsInXY=2` on PC high
- Profile with `profilegpu` — Water appears in `BasePass > WaterMesh`

---

## River Spline Setup

```markdown
1. Place AWaterBodyRiver in the level
2. Add spline points along the river path (minimum 2)
3. Set Width per spline point (river can narrow and widen)
4. Set Depth per spline point
5. For waterfalls: enable 'River To Ocean' connection on the terminal spline point
6. River mesh conforms to terrain when Landscape is below the spline
   — use Landscape Edit Mode to carve the river channel manually for clean banks
```

---

## Water Collision & Physics

```cpp
// Query water depth at a world location
float WaterDepth = 0.f;
FVector WaterSurfaceLocation, WaterSurfaceNormal;

if (UWaterBodyComponent* WaterBody = UWaterSubsystem::GetWaterBodyComponent(World, TestLocation))
{
    EWaterBodyQueryFlags QueryFlags = EWaterBodyQueryFlags::ComputeDepth
        | EWaterBodyQueryFlags::ComputeImmersionDepth;
    
    FWaterBodyQueryResult QueryResult = WaterBody->QueryWaterInfoClosestToWorldLocation(
        TestLocation, QueryFlags);

    if (QueryResult.IsInWater())
    {
        WaterDepth = QueryResult.GetWaterDepth();
        WaterSurfaceLocation = QueryResult.GetWaterSurfaceLocation();
        WaterSurfaceNormal = QueryResult.GetWaterSurfaceNormal();
    }
}
```

**Use case:** Determine if a character is wading, swimming, or underwater. `GetWaterDepth()` returns positive = submerged depth, 0 = at surface, negative = above water.

---

## Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| AWaterBodyOcean in streaming cell | Water surface disappears when cell unloads | Move to Always Loaded data layer |
| Water plugin Experimental on UE5.0–5.2 | Unstable mesh generation, visual glitches | Upgrade to UE5.3+ or accept instability |
| Too many pontoons | Physics tick cost dominates | Cap at 6 pontoons for small boats |
| Custom water material not using Single Layer Water domain | Incorrect depth fading, missing refraction | Set material domain to Single Layer Water |
| WaterInfoTexture manually assigned | Static texture, waves don't update | Remove assignment; plugin assigns it at runtime |
| River not conforming to Landscape | River floats above terrain | Carve Landscape channel manually; set River depth correctly |
