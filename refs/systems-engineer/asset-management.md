# Asset Management — UE5 Reference

## Hard vs Soft References

A **hard reference** (`UPROPERTY` with a direct `UStaticMesh*`, `UTexture2D*`, etc.) forces the referenced asset to load into memory whenever the owning class is loaded — including when the class is used as a `TSubclassOf` in a data table or spawned via `SpawnActor`. This creates invisible memory bloat that scales with how many actors reference large assets.

A **soft reference** (`TSoftObjectPtr<T>` / `TSoftClassPtr<T>`) stores the asset path only. The asset is loaded explicitly, on demand.

```cpp
// WRONG — this actor hard-references a 200MB texture; it loads any time the class loads
UPROPERTY(EditDefaultsOnly)
UTexture2D* HeroPortrait; // 200MB loaded on class load, even if never rendered

// CORRECT — soft reference; portrait loads only when explicitly requested
UPROPERTY(EditDefaultsOnly)
TSoftObjectPtr<UTexture2D> HeroPortrait;
```

### The Frequently-Spawned Actor Rule
Any actor that spawns frequently (projectiles, collectibles, enemies, pooled tiles) must use `TSoftObjectPtr` or `TSoftClassPtr` for large assets (meshes, textures, sound cues). Hard references on frequently-spawned actors multiply memory cost by instance count at load time.

---

## Async Asset Loading

### `FStreamableManager` — Direct Async Load

```cpp
// Declare in your subsystem or game instance (not as a local variable — it manages handles)
FStreamableManager StreamableManager;

// Request async load with callback
void UMyInventorySubsystem::LoadItemIcon(TSoftObjectPtr<UTexture2D> IconRef)
{
    StreamableManager.RequestAsyncLoad(
        IconRef.ToSoftObjectPath(),
        FStreamableDelegate::CreateUObject(this, &UMyInventorySubsystem::OnIconLoaded, IconRef)
    );
}

void UMyInventorySubsystem::OnIconLoaded(TSoftObjectPtr<UTexture2D> IconRef)
{
    if (UTexture2D* Icon = IconRef.Get()) // asset is now in memory
    {
        // Safe to use
    }
}
```

### `UAssetManager` — For Primary Assets and Bundles

Use `UAssetManager` when loading named primary assets (items, characters, abilities) defined in the Asset Manager settings:

```cpp
FPrimaryAssetId ItemId = FPrimaryAssetId("Item", "Sword_01");

UAssetManager::Get().LoadPrimaryAsset(
    ItemId,
    TArray<FName>{ FName("UI") }, // load bundle "UI" (icons, thumbnails)
    FStreamableDelegate::CreateUObject(this, &UMyItemSystem::OnItemLoaded, ItemId)
);
```

Asset bundles allow loading only the subset of an asset's references needed for a given context (e.g., "UI" bundle for menus, "Gameplay" bundle for in-world actors).

---

## `LoadSynchronous` — When It Is and Is Not Acceptable

`TSoftObjectPtr<T>::LoadSynchronous()` blocks the game thread until the asset is in memory. It is only acceptable in:

- Dedicated loading screens (player sees a load indicator)
- Editor utilities and commandlets
- One-time startup sequences where a single stall is acceptable

```cpp
// ACCEPTABLE — loading screen is active, stall is expected
UTexture2D* Tex = HeroPortrait.LoadSynchronous();

// NEVER in Tick, ability activation, or any runtime gameplay path
void AMyCharacter::Tick(float DeltaTime)
{
    // WRONG — stalls game thread; causes frame spike every time asset isn't cached
    UTexture2D* Tex = DynamicPortrait.LoadSynchronous();
}
```

---

## Checking Whether an Asset Is Already Loaded

Before triggering a load request, check if the asset is already in memory:

```cpp
if (HeroPortrait.IsValid())
{
    // Already loaded — use immediately
    UTexture2D* Tex = HeroPortrait.Get();
}
else if (!HeroPortrait.IsNull())
{
    // Path is set but not loaded — trigger async load
    LoadIconAsync(HeroPortrait);
}
// If IsNull() — no asset assigned, nothing to load
```

`IsValid()` = path set AND asset in memory. `IsNull()` = no path set. Neither check loads the asset.

---

## Reference Auditing — Finding Unexpected Hard References

Unexpected hard references are the most common source of excessive memory in UE5 projects. Use the Reference Viewer (`right-click asset → Reference Viewer`) to trace what a frequently-spawned actor is pulling into memory. Red connections in the viewer are hard references.

Common sources of accidental hard references:
- Blueprint class defaults referencing assets directly instead of using `TSoftObjectPtr`
- `TSubclassOf<>` properties on spawned actors that point to Blueprint classes with large asset references
- Particle systems or sound cues embedded directly in actor components instead of loaded on demand
