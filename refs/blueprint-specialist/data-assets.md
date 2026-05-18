# Data Assets & Tables — UE5 Reference

## Choosing the Right Data Container

| Container | Schema | Editor UX | Use case |
|---|---|---|---|
| `UPrimaryDataAsset` | Flexible — nested structs, arrays, soft refs | Details panel — full property editing | Per-object config: character def, ability def, item def |
| `UDataTable` | Fixed row struct — all rows same schema | Spreadsheet editor / CSV import | Tabular data: loot tables, dialogue, localization |
| `UCurveTable` | Float or vector curves per row | Curve editor per row | Damage falloff, animation curves, scaling tables |
| `UCurveFloat` / `UCurveVector` | Single curve | Curve editor | One-off value-over-time data |
| `UDataAsset` (base) | Flexible | Details panel | Simple, non-Asset-Manager-tracked config |

Use `UPrimaryDataAsset` when the asset needs Asset Manager registration, async loading, or bundle-based streaming. Use `UDataAsset` for simple config that does not require those features.

---

## UPrimaryDataAsset — Canonical Pattern

```cpp
// UCharacterDefinition.h
UCLASS(BlueprintType)
class MYGAME_API UCharacterDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // Required for Asset Manager — identifies asset type and unique name
    virtual FPrimaryAssetId GetPrimaryAssetId() const override
    {
        return FPrimaryAssetId(TEXT("CharacterDefinition"), GetFName());
    }

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Identity")
    FText DisplayName;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Stats",
        meta = (ClampMin = "1"))
    float BaseHealth = 100.f;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Stats")
    float BaseMovementSpeed = 600.f;

    // Soft reference — does NOT force-load the mesh when this asset loads
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Visuals")
    TSoftObjectPtr<USkeletalMesh> CharacterMesh;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Abilities")
    TArray<TSubclassOf<UGameplayAbility>> DefaultAbilities;

    // Nested soft-ref array — GAS tags, each a lightweight handle
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Tags")
    FGameplayTagContainer StartingTags;
};
```

---

## Asset Manager Registration

Register primary asset types in `DefaultGame.ini` so the Asset Manager indexes them at startup:

```ini
[/Script/Engine.AssetManagerSettings]
+PrimaryAssetTypesToScan=(
    PrimaryAssetType="CharacterDefinition",
    AssetBaseClass=/Script/MyGame.CharacterDefinition,
    bHasBlueprintClasses=False,
    bIsEditorOnly=False,
    Directories=((Path="/Game/Data/Characters")),
    Rules=(Priority=1,bApplyRecursively=True)
)
```

---

## Async Loading with Asset Manager (Blueprint)

Never hard-reference a `UPrimaryDataAsset` from an Actor constructor — it forces every referenced asset to load when the Actor class loads:

```
BAD — hard reference in Blueprint variable default:
[Variable: CharacterDef (UCharacterDefinition)]
Default Value: CharacterDef_Warrior  ← hard reference forces load at game start

GOOD — soft reference, async loaded when needed:
[Variable: CharacterDefRef (TSoftObjectPtr<UCharacterDefinition>)]
Default Value: CharacterDef_Warrior  ← stored as path only, not loaded

[Event BeginPlay]
  → [Async Load Primary Asset]
      PrimaryAssetId: [Get Primary Asset Id from CharDef soft ref]
      LoadBundles: ["UI", "Gameplay"]  ← load specific bundles
  → [On Loaded]
      → [Get Primary Asset Object] → [Cast to UCharacterDefinition]
      → [Apply Definition to Character]
```

---

## UDataTable — Tabular Data Pattern

Define the row struct first — all rows share this schema:

```cpp
// FItemTableRow.h
USTRUCT(BlueprintType)
struct FItemTableRow : public FTableRowBase
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Item")
    FText DisplayName;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Item")
    float Weight = 0.f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Item")
    int32 MaxStackSize = 1;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Item")
    TSoftObjectPtr<UTexture2D> Icon;
};
```

Create the DataTable asset in the editor: `Add New → Miscellaneous → Data Table → select FItemTableRow`.

### Reading a DataTable Row in Blueprint

```
[Get Data Table Row]
  Data Table: ItemDatabase (soft ref, loaded in advance)
  Row Name: "Sword_Iron"
  Out Row: FItemTableRow (Break Struct to access fields)
  Row Found: [branch on success]
```

### Reading in C++

```cpp
UPROPERTY(EditDefaultsOnly, Category = "Data")
TObjectPtr<UDataTable> ItemTable;

FItemTableRow* Row = ItemTable->FindRow<FItemTableRow>(FName("Sword_Iron"), TEXT("Item Lookup"));
if (Row)
{
    float ItemWeight = Row->Weight;
}
```

---

## Soft References — When and How

**Hard reference** (`TObjectPtr<T>`, direct asset property): asset loads when this object loads. Every Blueprint class that has a hard reference to an asset forces that asset to cook and load together.

**Soft reference** (`TSoftObjectPtr<T>`, `FSoftObjectPath`): stores a path string only. Asset loads only when explicitly requested.

| Use hard reference when | Use soft reference when |
|---|---|
| Asset is always needed when this object exists | Asset loads on demand (character skins, level-specific audio) |
| Asset is small (< 50KB) | Asset is large or varies by player choice |
| Asset is accessed in the first frame | Asset can afford a one-frame async load delay |

```cpp
// Blueprint-accessible soft reference property
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Visuals")
TSoftObjectPtr<UMaterialInterface> CharacterMaterial;

// C++ async load
void AMyCharacter::LoadMaterial()
{
    FStreamableManager& Streamable = UAssetManager::Get().GetStreamableManager();
    Streamable.RequestAsyncLoad(
        CharacterMaterial.ToSoftObjectPath(),
        FStreamableDelegate::CreateUObject(this, &AMyCharacter::OnMaterialLoaded));
}

void AMyCharacter::OnMaterialLoaded()
{
    if (UMaterialInterface* Mat = CharacterMaterial.Get())
    {
        GetMesh()->SetMaterial(0, Mat);
    }
}
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Hard reference to large asset in DataAsset | Everything that references the DataAsset loads the large asset | Switch to `TSoftObjectPtr` |
| Reading DataTable row by string literal | Typo causes silent miss — `Row Found` = false | Define row name constants in an enum or `FName` static |
| PrimaryDataAsset not registered in AssetManager | Asset Manager cannot find or async-load the asset | Add entry to `DefaultGame.ini PrimaryAssetTypesToScan` |
| Calling `GetPrimaryAssetId()` with wrong type string | Asset Manager indexes it under the wrong type — bundle loads fail | Match the type string exactly to what's in `DefaultGame.ini` |
| Struct field added to DataTable row without default | Existing rows get zero/empty value silently | Always add defaults to new FTableRowBase fields |
