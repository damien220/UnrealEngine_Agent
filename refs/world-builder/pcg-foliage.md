# PCG Foliage: Volumes, Nodes, Exclusion Zones & Runtime Generation

## PCG Volume vs. PCG World Actor

**`PCGVolume`** — Contained generation within a box. Easier to iterate in editor (move/resize volume, regenerate). Use for: small demo scenes, linear levels, dungeons, enclosed courtyard gardens.

**`PCGWorldActor`** — Global procedural generation. Required for World Partition and open worlds. Spans the entire world, generation respects World Partition cells (generates per-cell on demand). Use for: shipped open worlds, large-scale foliage scattering, ambient detail.

For production open worlds: **always use `PCGWorldActor`**. It integrates with streaming — foliage PCG nodes bake to static meshes per cell, unload when cell unloads.

## PCG Graph Nodes for Foliage

Core nodes for landscape foliage systems:

1. **Surface Sampler** — Generates grid of sample points on landscape surface. Inputs: landscape actor, grid density (points/100m²). Outputs: point list with normals. Foundation of all foliage scattering.

2. **Get Landscape Data** — Queries landscape properties: slope angle, height, layer weights. Filter by layer (e.g., only spawn trees where grass weight > 0.5).

3. **Static Mesh Spawner** — Takes point list, spawns instances. Outputs: Hierarchical Instanced Static Mesh Actor (HISMA). Can randomize mesh per point (LOD variants).

4. **Filter by Tag** — Input: point list and tag names. Output: filtered points matching tag criteria. Use with exclusion zones.

5. **Copy Points** — Clone input points N times with optional offset/rotation per copy. Useful for multi-layer foliage (shrubs + grass + flowers on same landscape patch).

6. **Transform Points** — Offset/rotate/scale points before spawning. Example: offset trees upward by Z to plant on hillslopes.

7. **Density Filter** — Reduce point count by density parameter. Input density 0.5 = spawn half the points. Useful for LOD or population control.

## Custom PCG Nodes in C++

For logic that Blueprint PCG graphs cannot express, subclass `UPCGSettings` in C++:

```cpp
// Header
#pragma once
#include "PCGNode.h"
#include "PCGSettings.h"
#include "MyCustomPCGNode.generated.h"

UCLASS()
class MYPROJECT_API UMyCustomPCGSettings : public UPCGSettings
{
    GENERATED_BODY()
public:
    // Pin properties
    virtual TArray<FPCGPinProperties> InputPinProperties() const override;
    virtual TArray<FPCGPinProperties> OutputPinProperties() const override;
    
    // Node metadata
    virtual FName GetDefaultNodeName() const override { return FName(TEXT("My Custom Node")); }
    virtual FText GetDefaultNodeTitle() const override { return FText::FromString(TEXT("My Custom Node")); }
    
    // Execution
    virtual FPCGElementPtr CreateElement() const override;
    
    // Custom parameters
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Settings")
    float CustomParameter = 1.0f;
};

// Implementation
TArray<FPCGPinProperties> UMyCustomPCGSettings::InputPinProperties() const
{
    TArray<FPCGPinProperties> Properties;
    FPCGPinProperties& InputPin = Properties.Emplace_GetRef();
    InputPin.Label = FName(TEXT("In"));
    InputPin.AllowedTypes = EPCGDataType::Point;
    InputPin.bAllowMultipleData = false;
    return Properties;
}

TArray<FPCGPinProperties> UMyCustomPCGSettings::OutputPinProperties() const
{
    TArray<FPCGPinProperties> Properties;
    FPCGPinProperties& OutputPin = Properties.Emplace_GetRef();
    OutputPin.Label = FName(TEXT("Out"));
    OutputPin.AllowedTypes = EPCGDataType::Point;
    return Properties;
}

// Custom element class
class FMyCustomPCGElement : public IPCGElement
{
public:
    virtual bool Execute(FPCGContext* Context) const override;
};

FPCGElementPtr UMyCustomPCGSettings::CreateElement() const
{
    return MakeShared<FMyCustomPCGElement>();
}

bool FMyCustomPCGElement::Execute(FPCGContext* Context) const
{
    // Read input points
    const TArray<FPCGTaggedData>& Inputs = Context->InputData.GetInputsByPin(FName(TEXT("In")));
    if (Inputs.IsEmpty()) return true;
    
    const FPCGTaggedData& Input = Inputs[0];
    const UPCGPointData* InputPoints = Cast<UPCGPointData>(Input.Data);
    if (!InputPoints) return true;
    
    // Create output
    UPCGPointData* OutputPoints = NewObject<UPCGPointData>();
    for (const FPCGPoint& Point : InputPoints->GetPoints())
    {
        FPCGPoint OutPoint = Point;
        // Your custom logic here: filter, transform, etc.
        OutputPoints->GetMutablePoints().Add(OutPoint);
    }
    
    // Output results
    FPCGTaggedData& Output = Context->OutputData.TaggedData.Emplace_GetRef();
    Output.Data = OutputPoints;
    Output.Pin = FName(TEXT("Out"));
    return true;
}
```

Register in module `.Build.cs`: `PublicDependencyModuleNames.Add("PCG");`

## Exclusion Zones: Spline-Based & Bounds-Based

**Spline exclusion approach**:

1. Create an actor with a `USplineComponent` (e.g., road path)
2. Tag the actor with `PCG_Exclusion` (standard UE convention)
3. In PCG graph, add **Get Spline Data** node (reads the spline)
4. Connect to **Difference** node (set subtraction of points)
5. Surface Sampler points − Spline points = filtered foliage

Runtime spline edits (e.g., road extends mid-game) automatically trigger PCG regeneration if the spline actor is a streaming source for PCG.

**Bounds-based exclusion**:

1. Place a volume (cube, sphere) around area to exclude
2. Use **Filter by Bounds** node with the volume's bounds
3. Connect to **Difference** node to subtract

**Example**: Roads exclude tree spawn, building footprints exclude foliage, water volumes exclude shrubs. Chain multiple Difference nodes for complex layouts.

## PCG & Nanite Budget: 16M Instance Limit

PCG graphs spawn Hierarchical Instanced Static Mesh (HISMA) actors. Each unique mesh = one Nanite virtual page table entry. Total Nanite budget: **16 million instances across the entire scene** (landscape detail, foliage, rocks, etc.).

Budget calculation:
```
Total instances = SUM of (cells loaded × density × instances per cell)
Example: 4km² open world, 256m cells = 16 cells loaded at 60fps
Foliage: 100 instances/cell × 16 = 1600 instances loaded
Buildings: 50 instances/cell × 16 = 800 instances loaded
Rocks: 200 instances/cell × 16 = 3200 instances loaded
Total: 5600 instances << 16M limit ✓
```

High-density foliage (> 1000 instances/km²) requires:
- Mesh simplification (lower tri count per instance)
- Nanite enablement on all foliage meshes
- Aggressive LOD (distant cells use simpler variants via PCG node branching)

## Pre-bake vs. Runtime Generation

**Pre-bake** (`PCG Generate` in editor): PCG graph bakes all output to static ISMs. Stored in level, no runtime cost. **Required for shipped games**. Storage: 1–10MB per km² depending on density.

**Runtime generation**: PCG graph executes at runtime (streaming source triggers graph execution in C++, not baked). **Only viable for < 1km² zones** — larger zones stall game thread (PCG is single-threaded). Use for: procedural dungeons, mission-specific layouts, time-limited event zones.

Production open worlds: **Always pre-bake**. Runtime generation only in exceptional cases (e.g., 50m² raid arena that's procedurally generated per run, never > 100 actors).

## Version Notes

- **UE5.1–5.2**: PCG framework foundation; blueprint nodes limited, custom C++ nodes recommended.
- **UE5.3+**: Expanded node library (Filter by Tag, Density Filter), better perf for large graphs.
- **UE5.4+**: PCG World Actor fully integrated with World Partition streaming; SpawnActors with HISMA batching optimized.

## Common Gotchas

1. **Multiple PCG Volumes overlap**: Same points spawn twice in overlap zone. Use Difference nodes or non-overlapping volume boundaries.
2. **Exclusion zone forgotten**: PCG spawns inside excluded area (e.g., trees in water). Check all exclusion zones are in graph before final bake.
3. **Nanite budget exceeded**: Too many instances, game stutters. Profile with `stat Nanite`, reduce density or simplify meshes.
4. **Custom node recompile crash**: Changes to `UPCGSettings` subclass cause old graph instances to hold stale pointers. Delete all PCG level instances, recompile, reimport.
5. **Pre-bake time explosion**: Dense foliage graph over large area can take 30+ minutes to bake. Bake in editor at night, or split world into regions with separate PCG bakes.

