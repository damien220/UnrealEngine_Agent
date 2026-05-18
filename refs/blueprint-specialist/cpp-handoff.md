# C++/Blueprint Handoff — Exposure Macros & Metadata Reference

## UFUNCTION Macro Decision Table

| Specifier | Execution pin | Return value | Blueprint can override | C++ default | Use when |
|---|---|---|---|---|---|
| `BlueprintCallable` | Yes | Yes | No | Only | Side-effecting function designers call |
| `BlueprintPure` | No | Yes | No | Only | Stateless getter, math helper — no side effects |
| `BlueprintImplementableEvent` | No execution pin in C++ | Yes | Yes (Blueprint required) | None | C++ fires it, Blueprint authors the entire response |
| `BlueprintNativeEvent` | No execution pin in C++ | Yes | Yes (Blueprint optional) | Yes (`_Implementation`) | C++ provides a fallback, Blueprint can override |

---

## BlueprintCallable — Side-Effecting Functions

```cpp
// Standard — execution pin, can return values
UFUNCTION(BlueprintCallable, Category = "Inventory",
    meta = (ToolTip = "Add item to inventory. Returns false if full."))
bool AddItem(TSubclassOf<UItemDefinition> ItemClass, int32 Quantity = 1);

// World context — allows calling from any Blueprint (like a static utility)
UFUNCTION(BlueprintCallable, Category = "Game",
    meta = (WorldContext = "WorldContextObject", ToolTip = "Get the current game phase."))
static EGamePhase GetCurrentGamePhase(UObject* WorldContextObject);

// Latent action (async, runs over multiple frames) — requires LatentInfo
UFUNCTION(BlueprintCallable, Category = "Loading",
    meta = (Latent, LatentInfo = "LatentInfo", ToolTip = "Async load a soft asset reference."))
void AsyncLoadAsset(TSoftObjectPtr<UObject> Asset, FLatentActionInfo LatentInfo);
```

---

## BlueprintPure — Stateless Reads

```cpp
// Const + BlueprintPure = no execution pin, evaluates on every read
UFUNCTION(BlueprintPure, Category = "Health",
    meta = (ToolTip = "Returns current health as a 0–1 percentage."))
float GetHealthPercent() const;

// Short math helpers benefit from CompactNodeTitle — cleaner node display in BP
UFUNCTION(BlueprintPure, Category = "Math",
    meta = (CompactNodeTitle = "LERP COLOR", ToolTip = "Linear interpolate between two colors."))
static FLinearColor LerpColor(FLinearColor A, FLinearColor B, float Alpha);
```

**Warning:** `BlueprintPure` functions are evaluated every time their output wire is read. If a designer connects the output to three nodes, the function runs three times. Cache results in a local variable for expensive Pure functions.

---

## BlueprintImplementableEvent — Blueprint Handles Everything

C++ declares the event; Blueprint authors the entire implementation. No C++ body defined.

```cpp
// C++ — declare only, no _Implementation in .cpp
UFUNCTION(BlueprintImplementableEvent, Category = "Ability",
    meta = (ToolTip = "Called when this ability activates. Implement visual/audio response in Blueprint."))
void OnAbilityActivated(const FGameplayAbilityActivationInfo& ActivationInfo);

UFUNCTION(BlueprintImplementableEvent, Category = "Interaction",
    meta = (ToolTip = "Called when an Actor enters this trigger zone."))
void OnActorEntered(AActor* EnteredActor);
```

Use when: the C++ system fires a lifecycle event and the only meaningful response is visual/audio — there is no useful C++ default behavior.

---

## BlueprintNativeEvent — C++ Default, Blueprint Can Override

C++ provides a default implementation; Blueprint can override it. Requires `_Implementation` in .cpp.

```cpp
// .h
UFUNCTION(BlueprintNativeEvent, Category = "Damage",
    meta = (ToolTip = "Determine if this actor can receive damage of the given type. Default: true."))
bool CanReceiveDamage(EDamageType DamageType) const;
virtual bool CanReceiveDamage_Implementation(EDamageType DamageType) const;

// .cpp — default fallback
bool AMyActor::CanReceiveDamage_Implementation(EDamageType DamageType) const
{
    return true; // Override in Blueprint for specific actor types
}

// Calling from C++ — always use the virtual dispatch path
bool bCanTakeDamage = Execute_CanReceiveDamage(TargetActor, DamageType);
// Never call CanReceiveDamage_Implementation directly — it bypasses Blueprint overrides
```

---

## UPROPERTY Exposure

```cpp
// EditDefaultsOnly — designer configures in Class Defaults, not per-instance
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Character|Stats",
    meta = (ClampMin = "0.0", UIMin = "0.0", ToolTip = "Base health at spawn."))
float BaseHealth = 100.f;

// EditAnywhere — per-instance override allowed in level editor
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Character|Team")
ETeam DefaultTeam = ETeam::None;

// Transient — runtime state, not saved, not shown in Details panel
UPROPERTY(Transient, BlueprintReadOnly, Category = "Character|Runtime")
float CurrentHealth = 0.f;

// VisibleAnywhere — shows value in Details panel but not editable by designer
UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Components")
TObjectPtr<UHealthComponent> HealthComponent;
```

---

## Metadata Specifiers Reference

| Specifier | Effect | Example |
|---|---|---|
| `Category = "X\|Y"` | Organizes in Blueprint palette — pipe-delimited hierarchy | `Category = "Inventory\|Items"` |
| `ToolTip = "..."` | Tooltip text shown in Blueprint editor hover | `meta = (ToolTip = "Returns current HP 0–MaxHP.")` |
| `ClampMin / ClampMax` | Clamps numeric input in Details panel | `meta = (ClampMin = "0", ClampMax = "1")` |
| `UIMin / UIMax` | Slider range in Details panel (does not enforce at runtime) | `meta = (UIMin = "0", UIMax = "100")` |
| `CompactNodeTitle` | Short display name on the Blueprint node | `meta = (CompactNodeTitle = "HP%")` |
| `DisplayName` | Override the palette/node display name | `meta = (DisplayName = "Add Item to Inventory")` |
| `Keywords` | Extra search terms in Blueprint palette search | `meta = (Keywords = "pickup loot inventory")` |
| `WorldContext` | Marks which parameter is the world context for library functions | `meta = (WorldContext = "WorldContextObject")` |
| `HidePin` | Hides a parameter from the Blueprint node | `meta = (HidePin = "WorldContextObject")` |
| `ExpandEnumAsExecs` | Turns enum return value into multiple exec output pins | `meta = (ExpandEnumAsExecs = "ReturnValue")` |

---

## Blueprint Function Library — Stateless Utility Pattern

```cpp
// UMyGameplayLibrary.h
UCLASS()
class MYGAME_API UMyGameplayLibrary : public UBlueprintFunctionLibrary
{
    GENERATED_BODY()

public:
    // Pure math / data utility — no world context needed
    UFUNCTION(BlueprintPure, Category = "Gameplay|Math",
        meta = (ToolTip = "Calculate damage after applying armor reduction."))
    static float CalculateArmorReduction(float RawDamage, float ArmorValue);

    // World-aware utility — WorldContext allows calling from any Actor
    UFUNCTION(BlueprintCallable, Category = "Gameplay|Actors",
        meta = (WorldContext = "WorldContextObject", HidePin = "WorldContextObject",
                DefaultToSelf = "WorldContextObject",
                ToolTip = "Find nearest actor of class within radius."))
    static AActor* FindNearestActorOfClass(
        UObject* WorldContextObject,
        TSubclassOf<AActor> ActorClass,
        FVector Origin,
        float Radius);
};
```

`DefaultToSelf` + `HidePin` on the WorldContext parameter makes the call site clean — the world context is auto-filled from the calling actor.

---

## UINTERFACE — Interface Declaration Pattern

```cpp
// UMyDamageable.h
UINTERFACE(MinimalAPI, Blueprintable,
    meta = (ToolTip = "Implemented by Actors that can receive damage."))
class UMyDamageable : public UInterface { GENERATED_BODY() };

class MYGAME_API IMyDamageable
{
    GENERATED_BODY()

public:
    // BlueprintNativeEvent — C++ default exists, Blueprint can override
    UFUNCTION(BlueprintCallable, BlueprintNativeEvent, Category = "Damage")
    void ApplyDamage(float Amount, EDamageType DamageType, AActor* DamageInstigator);

    // BlueprintImplementableEvent — no C++ default, Blueprint required
    UFUNCTION(BlueprintCallable, BlueprintImplementableEvent, Category = "Damage")
    void OnDamageReceived(float Amount, EDamageType DamageType);
};
```

Call via `Execute_` wrappers from C++; call via the interface message node in Blueprint. Never `Cast To` for interface calls — use `Does Implement Interface` then the message node.
