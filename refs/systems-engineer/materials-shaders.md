# Materials & Shaders — Reference

## Overview

UE5's material system compiles HLSL shaders from visual node graphs. Understanding when to use Custom nodes, Material Functions, Material Parameter Collections, and Runtime Virtual Textures determines both rendering correctness and GPU performance. Material decisions made early (hierarchy depth, RVT adoption, masked-on-Nanite) are expensive to change later.

---

## Custom HLSL Expression Nodes

The `Custom` material expression node embeds hand-written HLSL. It is a last resort — prefer built-in nodes because Custom nodes bypass optimization passes.

### Hard restrictions
| Restriction | Reason |
|---|---|
| No texture samplers without manual `Texture2D` / `SamplerState` input pins | Sampler slots are GPU-limited; undeclared samplers compile silently but fail on some platforms |
| No dynamic branching on per-pixel data | UE's shader compiler cannot eliminate dead paths — performance equals both paths executed |
| No global `static` variables inside the node | Each node instance is inlined; statics cause multiply-defined-symbol compile errors |
| No `#include` of external files | Custom nodes compile in isolation; only UE's built-in HLSL intrinsics are available |

### Multiple return values via `UseCustomOutputs`
```hlsl
// Add output pins in node settings: OutAlpha (float), OutEmissive (float3)
// In code:
OutAlpha = saturate(sin(Time));
OutEmissive = float3(1, 0.5, 0) * OutAlpha;
return float3(0, 0, 0); // Unused primary output when using custom outputs
```

### When to use Custom node vs alternatives
| Goal | Use |
|---|---|
| Complex math expressible in nodes | Built-in nodes (no compile risk) |
| Reusable shader logic, no platform risk | Material Function (compiled inline, zero overhead) |
| One-off algorithm not in node set | Custom node |
| Global per-frame scalar or vector | Material Parameter Collection (UBO, one GPU upload per frame) |

---

## Material Functions vs Material Parameter Collections

### Material Functions
- Compiled **inline** at the call site — no draw-call overhead, no indirection
- Encapsulate node graphs as reusable assets (`Engine/Content` or project `Materials/Functions/`)
- Ideal for: noise generators, triplanar mapping, layer blending logic, PBR utilities
- Use `Function Input` / `Function Output` nodes inside the function; expose inputs with Preview Values for editor visualization

```
Parent Material → Material Function (inline) → same GPU cost as writing nodes directly
```

### Material Parameter Collections (MPC)
- UBO (Uniform Buffer Object) — one CPU→GPU upload per frame regardless of how many materials reference it
- Best for: time-of-day scalars, global wind direction, weather wet-factor, player proximity effect radius
- **Never** drive per-instance data with MPC — MPC has no per-instance distinction
- Access in C++: `UKismetMaterialLibrary::SetScalarParameterValue(World, MPC, "Time", GameTime)`
- Max 1024 scalar parameters + 256 vector parameters per MPC asset

```cpp
// C++ — update MPC once per frame (e.g., in GameState Tick)
#include "Kismet/KismetMaterialLibrary.h"

void AMyGameState::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    if (IsValid(WorldMPC))
    {
        UKismetMaterialLibrary::SetScalarParameterValue(
            this, WorldMPC, TEXT("TimeOfDay"), CurrentTimeOfDay);
    }
}
```

---

## Material Instance Hierarchy

### Rules
- `UMaterialInterface` → `UMaterial` (base, never assign directly to meshes) → `UMaterialInstance` (static) → `UMaterialInstanceDynamic` (runtime mutable)
- Never skip levels: a `UMaterialInstance` must derive from a `UMaterial` or another `UMaterialInstance`, never from a `UMaterialInstanceDynamic`
- Dynamic Material Instances have a CPU cost per parameter change — cache them, never create per-frame

### Creating and caching a Dynamic Material Instance
```cpp
// WRONG — creates a new UMaterialInstanceDynamic every frame
void AMyActor::Tick(float DeltaTime)
{
    UMaterialInstanceDynamic* MID = UMaterialInstanceDynamic::Create(BaseMaterial, this);
    MID->SetScalarParameterValue(TEXT("EmissiveStrength"), EmissiveValue); // Leaks!
}

// CORRECT — create once in BeginPlay, cache, reuse
UPROPERTY()
TObjectPtr<UMaterialInstanceDynamic> DynamicMaterial;

void AMyActor::BeginPlay()
{
    Super::BeginPlay();
    UStaticMeshComponent* Mesh = GetComponentByClass<UStaticMeshComponent>();
    if (IsValid(Mesh))
    {
        DynamicMaterial = Mesh->CreateDynamicMaterialInstance(0); // slot 0
    }
}

void AMyActor::SetGlow(float Intensity)
{
    if (IsValid(DynamicMaterial))
    {
        DynamicMaterial->SetScalarParameterValue(TEXT("EmissiveStrength"), Intensity);
    }
}
```

### Setting parameters at runtime
| Function | Use |
|---|---|
| `SetScalarParameterValue(Name, float)` | Floats, 0–1 masks, blend weights |
| `SetVectorParameterValue(Name, FLinearColor)` | Colors, directions, HDR values |
| `SetTextureParameterValue(Name, UTexture*)` | Swap texture at runtime (e.g., team color decal) |

---

## Runtime Virtual Textures (RVT)

RVT eliminates per-pixel layer blending overhead in secondary materials (roads, foliage, debris) by baking Landscape material output into a virtual texture that other objects sample.

### Setup — Landscape side

1. Create a `URuntimeVirtualTexture` asset. Set **Virtual Texture Size** (e.g., 4096), **Tile Size**, and **Content** (`BaseColor + Normal` or `BaseColor + Normal + Roughness`)
2. Place an `ARuntimeVirtualTextureVolume` actor to cover the terrain; assign the `URuntimeVirtualTexture` asset to it
3. In the **Landscape material**, add a `RuntimeVirtualTextureOutput` node. Wire `Base Color`, `Specular`, `Roughness`, and `Normal` from the main shading path into it
4. Enable **Virtual Texture** on the Landscape actor (`Details > Virtual Texture`)

### Setup — Receiving material side (foliage, roads, rocks)
```
Add RuntimeVirtualTextureSample node:
  - Virtual Texture: [assign the same URuntimeVirtualTexture asset]
  - Outputs: BaseColor, Specular, Roughness, WorldNormal (select which are needed)
  - Blend with mesh's own detail texture using a Lerp driven by a blend mask
```

### Performance benefit
- Without RVT: every foliage material performs full Landscape layer blend per pixel
- With RVT: foliage reads one virtual texture lookup (similar cost to a regular texture sample)
- Typical gain: 0.5–2ms GPU on terrain-heavy scenes

### Gotchas
- RVT capture only updates when the Landscape or bound actors change — not per-frame by default
- Mobile and some consoles have limited RVT support — check platform feature level

---

## Nanite Materials

Nanite only renders opaque, non-WPO geometry by default. Deviations carry penalties.

| Material type | Nanite behavior | Cost |
|---|---|---|
| Fully opaque, no WPO | Full Nanite rasterization | Baseline |
| Masked (opacity mask) | Two-pass: Nanite rasterizes, then masked pass re-renders visible pixels | ~1.5–2× draw cost on masked surfaces |
| Translucent | Falls back to traditional rasterizer; **Nanite not used** | Full traditional mesh cost |
| WPO (World Position Offset) | Supported UE5.1+ with `r.OptimizedWPO 1`; requires recompute of bounds | Slight overhead for bound update |
| Spline / Procedural Mesh | Nanite unsupported | Traditional mesh |
| Skeletal mesh | Nanite unsupported | Traditional mesh |

```
Profiling masked Nanite meshes: stat Nanite → look for Masked draw count.
r.Nanite.AllowMaskedMaterials 0  ← temporarily disable to measure perf difference
```

---

## Custom Depth & Stencil (Outline / Highlight Effects)

Used for silhouette outlines (selected objects, enemy highlights) via post-process.

### Setup
1. Enable **Custom Depth-Stencil** in `Project Settings > Rendering > Custom Depth-Stencil Pass` → set to `Enabled with Stencil`
2. On the target mesh: `Rendering > Render Custom Depth Pass = true`, `Custom Depth Stencil Value = N` (1–255)
3. In a **Post Process Material** (domain = `Post Process`):
   - Sample `SceneTexture:CustomDepth` and `SceneTexture:CustomStencil`
   - Compare stencil value to detect outlined objects
   - Use a Sobel edge detection kernel on CustomDepth for a 1-pixel outline

```
r.CustomDepth.Order 1   ← render custom depth before main pass (lower latency on outline)
r.CustomDepth.Order 0   ← render after main pass (default, avoids overdraw on non-outlined frames)
```

### Performance note
Custom Depth is an extra render pass — only enable `Render Custom Depth Pass` on meshes that need it, not globally.

---

## Profiling & Optimization

| Tool / Command | What it measures |
|---|---|
| `ProfileGPU` (console command) | Per-pass GPU time; expands to `BasePass`, `ShadowDepths`, `Translucency`, etc. |
| `stat RHI` | Raw RHI draw calls and triangle count per frame |
| `r.MaterialQualityLevel 0/1/2/3` | Switch quality level at runtime (Low / Medium / High / Epic) |
| `stat Nanite` | Nanite rasterized vs fallback triangle counts |
| `r.ShaderComplexity 1` | Heatmap overlay: red = expensive, green = cheap |
| Shader Permutation audit | `Edit > Project Settings > Rendering > Shader Permutation Reduction` — disable unused feature combos |

### Reducing shader permutations
- Use `Static Switch` parameters conservatively — each unique combination is a separate shader permutation
- Use `Quality Switch` nodes instead of manual `StaticSwitch` for LOD-driven quality (UE manages permutations automatically)
- Disable unused `r.SupportAllShaderPermutations` combinations in Project Settings for shipping builds

---

## Module Dependencies

No additional `.Build.cs` dependency is required for material-related C++ API. Headers are in `Engine`:

```cpp
#include "Materials/MaterialInstanceDynamic.h"     // UMaterialInstanceDynamic
#include "Materials/MaterialParameterCollection.h"  // UMaterialParameterCollection
#include "Kismet/KismetMaterialLibrary.h"           // UKismetMaterialLibrary::SetScalarParameterValue
#include "Engine/Texture2D.h"                       // UTexture2D (for SetTextureParameterValue)
```
