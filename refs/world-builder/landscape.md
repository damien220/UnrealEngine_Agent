# Landscape Resolution, Layers & Runtime Virtual Texturing

## Landscape Resolution Formula & Component Sizing

Landscape dimensions are **`(ComponentsX * ComponentSize * QuadsPerSection) + 1`** vertices in each direction. Example:

- 10×10 components, 63×63 quads per component (1 section) = (10 * 63) + 1 = **631×631 vertices** (630×630 quads)
- 32×32 components with 4 subsections (126×126 quads per component) = (32 * 126) + 1 = **4033×4033 vertices** (4032×4032 quads)

Valid landscape sizes follow the pattern: **(2^n - 1) or (2^n * 2 - 2)**. Do not use arbitrary resolutions; the import process will silently pad to the nearest valid size, wasting memory.

Common valid sizes (vertices):
- 64×64 (63×63 quads, smallest)
- 129×129 (128×128 quads)
- 257×257 (256×256 quads)
- 513×513 (512×512 quads)
- 1009×1009 (1008×1008 quads, typical for open worlds)
- 2017×2017 (2016×2016 quads, very large)

### Component vs. Section

- **Component** = one unit of Landscape, always square, has fixed memory cost (heightmap, weights, rendering)
- **Section** = subdivision within a component for LOD purposes (1 or 4 subsections per component)
- **QuadsPerSection** = vertices - 1 per section (power of 2, typically 64×64 or 128×128 vertices = 63 or 127 quads)

Rule of thumb: Epic recommends **max 1024 components** per landscape (32×32 or smaller) for manageable CPU costs. Larger component counts increase render-thread overhead.

### Heightmap Import Format

Import heightmaps as **R16** (16-bit unsigned, standard) or **R32** (32-bit float, for extreme height variation). Z scale is applied during import: `final_height = raw_heightmap_value * Z_scale`. Store your source heightmaps in these formats to avoid precision loss.

## Landscape Material Layers & Blending

Landscape materials use the **`LandscapeLayerBlend`** node to blend multiple painted layers. Three blend modes:

1. **`LB Weight Blend`** — Non-order-dependent weighted blending. All LB Weight Blend layers are combined proportionally based on painted weights (0–1). Use when surfaces should blend smoothly without layering hierarchy (e.g., grass → dirt transition).

2. **`LB Height Blend`** — Weight-based blending with **height-based detail offset**. Each layer has a height value (from texture alpha channel) that determines blending boundary. Higher-height layers "sit on top" of lower layers at boundaries. Use when one material should clearly sit atop another (e.g., snow over rock). **Important**: If mixing LB Height Blend with other blend modes, set the base layer to `LB Alpha Blend` and others to `LB Height Blend` to avoid black artifacts at transitions.

3. **`LB Alpha Blend`** — Order-dependent alpha overlay on top of Weight/Height layers. Applied in list order, each fully opaque at blended alpha. Use for decals, stains, or final detail layers.

To create a paintable layer: (a) create a **`Landscape Layer Info`** asset (Editor > Create > Landscape Layer Info), (b) assign it to the landscape in `Landscape Settings > Paint > Target Layers`, (c) reference it in the material's `LandscapeLayerBlend` node. If a layer is referenced in the material but no Layer Info exists, its weight is always zero (invisible).

### Maximum Visible Layers Per Region

No hard limit documented, but **performance degrades with > 4–8 layer samples per pixel** due to texture sampler costs. Mobile (ES3.1) has a hard cap of **16 texture samplers total** across the entire material — landscape-heavy materials for mobile should use ≤ 4 layers. Desktop can comfortably support 8–12 layers if the material is optimized.

## Runtime Virtual Texturing on Landscape

RVT creates GPU-rendered virtual texture pages on demand, eliminating expensive per-pixel layer blending for secondary materials (rocks, roads, decals). Setup:

### Step 1: Create RVT Asset

1. In Content Drawer, create `Runtime Virtual Texture` (Textures category)
2. Set tile size (typically 128×128 or 256×256 pixels)
3. Set virtual texture size (typically 8k or 16k)
4. Enable "Render Target RGB Output" if capturing color

### Step 2: Landscape Material Setup

1. Open Landscape material, enable **`Use Material Attributes`** (Details > Material)
2. Add **`Make Material Attributes`** expression
3. Connect all layer outputs (Base Color, Normal, Roughness, etc.) to it
4. Add **`Runtime Virtual Texture Output`** node
5. Connect the Make Material Attributes output to the RVT Output
6. Ensure RVT Output's `Virtual Texture Output` property names the RVT asset you created

### Step 3: Place RVT Volume

1. In Level, place `Runtime Virtual Texture Volume` (Place Actor > Volumes)
2. Set its `Virtual Texture` to your RVT asset
3. Scale the volume to cover your entire landscape (use "Transform from Bounds" → select Landscape → auto-fit)

### Step 4: Enable Landscape to Render to RVT

1. Select Landscape actor
2. Details > Virtual Textures > + Add Element
3. Assign your RVT asset
4. Landscape now renders to RVT every frame

### Step 5: Receiving Objects Sample RVT

Materials on secondary objects (decals, splines, landscape-aligned rocks):

1. Add **`Runtime Virtual Texture Sample`** node
2. Point it to your RVT asset
3. Sample outputs feed into their respective material attributes (Base Color → Base Color pin, etc.)
4. Enable those objects' "Draw in Virtual Textures" property with the same RVT asset

**Performance gain**: Receiving materials no longer evaluate the expensive landscape material. Instead, they sample a single RVT texture, reducing shader cost by 10–50x.

## Landscape Holes & Deletion

**Correct approach**: Use a **Visibility Layer** to create holes (paint black on visibility layer = invisible). Visibility layer is free — it's just a special weightmap.

**Wrong approach**: Deleting landscape components to make holes breaks: (a) water/fluid simulation across the boundary, (b) LOD transitions (visible seams), (c) collision assumptions. **Never delete components to create holes.**

## LOD Configuration & Streaming

Landscape LOD is calculated per subsection (smallest unit). Configure in `Landscape Settings > LOD`:

- **Num LODs**: Max LOD levels generated (typically 1–2 for tight LOD, 4+ for extreme far-distance).
- **LOD Distance Factor**: Multiplier for LOD distance. Higher = aggressive LOD (switch to lower detail sooner). Typical: 1.0–2.0.

For streaming large landscapes (World Partition), use:

- **`r.LandscapeLOD.DistributionSetting`** (console): 0 = uniform LOD, 1 = camera-distance-based (default, better for open worlds)
- **Streaming Pool**: Configure `r.Streaming.PoolSize` to ensure landscape textures fit. Exhausted pool = blurry terrain.

Test with `stat Streaming` and Unreal Insights `AssetLoading` trace to profile landscape texture loads. If hitches occur when landscape loads, reduce cell size or spread LOD transitions across more distance.

## Version Notes

- **UE5.0–5.1**: Landscape materials are standard; RVT support requires manual setup.
- **UE5.2+**: "Landscape Height and Layer Biasing" improved; RVT performance optimized.
- **UE5.3+**: Landscape LOD and World Partition streaming fully integrated, no separate config needed.

## Common Gotchas

1. **Import padding**: Importing a 512×512 heightmap results in 513×513 (padded to nearest valid size). Pre-calculate your source heightmap to match valid landscape dimensions.
2. **Height precision**: Z scale of 100 with R16 import = 0.00381 cm per unit (very fine). Z scale of 1 with R32 = 0.0001 precision. Choose wisely based on terrain detail needed.
3. **Layer Info mismatch**: Layer referenced in material but Layer Info doesn't exist → weight always 0 (invisible layer). Check Details > Landscape > Paint Layers to confirm Layer Info is registered.
4. **RVT memory**: RVT pools memory per frame. 16k RVT with 256×256 tile = ~1GB VRAM. Large landscapes may need multiple RVT assets per region to stay within memory budgets.
5. **Component count bloat**: Many small components (< 50m² per component) inflate CPU costs. Consolidate to larger components if terrain is uniform.

