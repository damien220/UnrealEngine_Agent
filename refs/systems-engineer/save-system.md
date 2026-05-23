# Save System — Reference

## Overview

UE5's save system is built around `USaveGame` subclasses serialized via `UGameplayStatics`. It is intentionally simple — no database, no built-in versioning. Any project beyond a prototype needs async save/load, a version migration strategy, and a clear policy on which state belongs in `USaveGame` vs `UGameInstance` vs `UPlayerState`.

---

## USaveGame Pattern

### Defining a save game object
```cpp
// MySaveGame.h
#include "GameFramework/SaveGame.h"
#include "MySaveGame.generated.h"

UENUM(BlueprintType)
enum class ESaveDataVersion : uint8
{
    Initial       = 0,
    AddedInventory = 1,
    AddedSettings  = 2,
    Latest         = AddedSettings
};

UCLASS()
class MYGAME_API UMySaveGame : public USaveGame
{
    GENERATED_BODY()
public:
    // Version field — always save first
    UPROPERTY(SaveGame)
    ESaveDataVersion SaveVersion = ESaveDataVersion::Latest;

    // Gameplay progress
    UPROPERTY(SaveGame)
    int32 PlayerLevel = 1;

    UPROPERTY(SaveGame)
    float PlaytimeSeconds = 0.f;

    UPROPERTY(SaveGame)
    FString LastCheckpointTag;

    // Inventory — added in version AddedInventory
    UPROPERTY(SaveGame)
    TArray<FName> InventoryItemIDs;

    // Settings — added in version AddedSettings
    UPROPERTY(SaveGame)
    float MasterVolume = 1.f;

    UPROPERTY(SaveGame)
    int32 GraphicsQuality = 2;
};
```

**Rule**: Every field that must be saved across sessions requires `UPROPERTY(SaveGame)`. Fields without this specifier are skipped by the serializer even if they have `UPROPERTY()`.

### Saving (synchronous)
```cpp
#include "Kismet/GameplayStatics.h"

void UMySaveSubsystem::SaveGame(const FString& SlotName)
{
    UMySaveGame* SaveObj = Cast<UMySaveGame>(
        UGameplayStatics::CreateSaveGameObject(UMySaveGame::StaticClass()));

    // Populate from live state
    SaveObj->PlayerLevel = GetPlayerLevel();
    SaveObj->PlaytimeSeconds = GetTotalPlaytime();

    // Synchronous save — blocks main thread; avoid on large saves
    bool bSuccess = UGameplayStatics::SaveGameToSlot(SaveObj, SlotName, 0 /*UserIndex*/);
}
```

### Loading (synchronous)
```cpp
UMySaveGame* UMySaveSubsystem::LoadGame(const FString& SlotName)
{
    if (!UGameplayStatics::DoesSaveGameExist(SlotName, 0))
    {
        return nullptr;
    }

    UMySaveGame* SaveObj = Cast<UMySaveGame>(
        UGameplayStatics::LoadGameFromSlot(SlotName, 0));

    if (!IsValid(SaveObj)) return nullptr;

    // Run migration before using the data
    MigrateIfNeeded(SaveObj);
    return SaveObj;
}
```

---

## Async Save / Load

Use async for any save file larger than a few KB — synchronous save on large files produces a visible hitch.

```cpp
// Async save
void UMySaveSubsystem::AsyncSave(const FString& SlotName, UMySaveGame* SaveObj)
{
    FAsyncSaveGameToSlotDelegate SaveDelegate;
    SaveDelegate.BindUObject(this, &UMySaveSubsystem::OnSaveComplete);
    UGameplayStatics::AsyncSaveGameToSlot(SaveObj, SlotName, 0, SaveDelegate);
}

void UMySaveSubsystem::OnSaveComplete(const FString& SlotName, int32 UserIndex, bool bSuccess)
{
    if (!bSuccess)
    {
        UE_LOG(LogMyGame, Error, TEXT("Save failed: slot=%s"), *SlotName);
    }
}

// Async load
void UMySaveSubsystem::AsyncLoad(const FString& SlotName)
{
    FAsyncLoadGameFromSlotDelegate LoadDelegate;
    LoadDelegate.BindUObject(this, &UMySaveSubsystem::OnLoadComplete);
    UGameplayStatics::AsyncLoadGameFromSlot(SlotName, 0, LoadDelegate);
}

void UMySaveSubsystem::OnLoadComplete(
    const FString& SlotName, int32 UserIndex, USaveGame* SaveGame)
{
    UMySaveGame* Loaded = Cast<UMySaveGame>(SaveGame);
    if (!IsValid(Loaded))
    {
        UE_LOG(LogMyGame, Warning, TEXT("Save slot not found or corrupt: %s"), *SlotName);
        return;
    }
    MigrateIfNeeded(Loaded);
    ApplySaveData(Loaded);
}
```

---

## Save Versioning

### Migration pattern
```cpp
void UMySaveSubsystem::MigrateIfNeeded(UMySaveGame* Save)
{
    if (Save->SaveVersion >= ESaveDataVersion::Latest) return;

    // Each case falls through — order matters
    switch (Save->SaveVersion)
    {
    case ESaveDataVersion::Initial:
        // Version 0 → 1: InventoryItemIDs didn't exist; default to empty (already is)
        [[fallthrough]];

    case ESaveDataVersion::AddedInventory:
        // Version 1 → 2: MasterVolume didn't exist; default to 1.0 (already is)
        [[fallthrough]];

    default:
        break;
    }

    Save->SaveVersion = ESaveDataVersion::Latest;
}
```

### Versioning rules
- **Never rename or remove** a `UPROPERTY(SaveGame)` field — the binary serializer matches fields by name. Rename = that field reads as default value on old saves silently.
- **Never change** the type of a saved field between versions (e.g., `int32` → `float`) — undefined serialization behavior.
- To rename a field: add the new field, migrate the value from the old field in `MigrateIfNeeded`, mark the old field `DEPRECATED` (keep in UPROPERTY so old saves can still deserialize it).
- Always increment `SaveVersion` and add a migration case before shipping.

---

## State Ownership — What Lives Where

| State type | Owner | Persistence |
|---|---|---|
| Player progress, inventory, settings | `USaveGame` | Cross-session (on disk) |
| Runtime session data (party, lobby state) | `UGameInstance` | Current process lifetime |
| Per-player in-session replicated state | `UPlayerState` | Current session, resets on rejoin |
| World state (destructibles, flags) | `USaveGame` + custom serialization | On disk |
| Transient gameplay state (current health) | `AActor` / `UAttributeSet` | Not saved — derived from save data on load |

**Key rule**: Do not save actor pointers or object references — they are invalid after a level unload or session restart. Save data by value (IDs, names, structs) and reconstruct actors from saved data in BeginPlay.

---

## Slot Management

```cpp
// Check if slot exists
bool bExists = UGameplayStatics::DoesSaveGameExist(TEXT("SaveSlot_0"), 0);

// Delete a slot
UGameplayStatics::DeleteGameInSlot(TEXT("SaveSlot_0"), 0);

// Naming convention: SlotName is arbitrary — use a consistent pattern
// Single save:    "MainSave"
// Multiple slots: "SaveSlot_0", "SaveSlot_1", "AutoSave"
// Per-player (console split-screen): "SaveSlot_{UserIndex}"
```

### UserIndex
- **PC (single player)**: always `0`
- **Console split-screen**: `0`–`3` matching player controller index. Each player has their own save storage on console; using the wrong index writes to the wrong player's save
- `UGameplayStatics::GetPlatformUserName(UserIndex)` retrieves the signed-in account name for the given index

---

## Saving Custom Structs

```cpp
// Any struct stored in USaveGame must be a USTRUCT
USTRUCT()
struct FItemRecord
{
    GENERATED_BODY()

    UPROPERTY(SaveGame)
    FName ItemID;

    UPROPERTY(SaveGame)
    int32 Quantity = 1;

    UPROPERTY(SaveGame)
    float Durability = 1.f;
};

// In the save game class:
UPROPERTY(SaveGame)
TArray<FItemRecord> Inventory;
```

Structs do not need `UCLASS` — they use `USTRUCT` and the `GENERATED_BODY()` macro. All fields that need to persist require their own `UPROPERTY(SaveGame)`.

---

## Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Saving a `UObject*` pointer | Null on load; object no longer exists | Save an ID (FName, FString) and reconstruct |
| Saving an actor reference | Crashes or null after level reload | Save spawn data (class, transform, state), respawn on load |
| Missing `SaveGame` specifier on UPROPERTY | Field never serialized; always reads default | Add `UPROPERTY(SaveGame)` |
| Synchronous save on large file | 50–200ms frame hitch | Use `AsyncSaveGameToSlot` |
| Renaming a UPROPERTY field | Old saves read that field as zero/default | Keep old field, add new, migrate in `MigrateIfNeeded` |
| No version field | Cannot migrate old saves safely | Always store `SaveVersion` as first field |
| Not checking `DoesSaveGameExist` before load | `LoadGameFromSlot` returns null; unchecked null crashes | Always check existence first |
