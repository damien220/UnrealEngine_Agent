# Architecture Patterns — UE5 Reference

## Composition Over Deep Inheritance

### The Rule
Prefer `UActorComponent` subclasses over deep `AActor` inheritance chains. A character with five components is maintainable at 10 engineers and 3 years. A 10-level `ACharacter` subclass hierarchy is not.

```
WRONG — inheritance explosion
ABaseCharacter
  └── APlayerCharacter
        └── AArmedPlayerCharacter
              └── AArmedPlayerCharacterWithShield
                    └── AArmedPlayerCharacterWithShieldAndJetpack

CORRECT — composition
ACharacter
  + UHealthComponent
  + UInventoryComponent
  + UAbilitySystemComponent
  + UEquipmentComponent
  + UMovementExtensionComponent
```

### When Inheritance Is Correct
One level of inheritance for the engine contract (e.g., your project's `ABaseCharacter : public ACharacter`) is fine. Two levels for a meaningful gameplay distinction (e.g., `AEnemyCharacter : public ABaseCharacter`) is acceptable. Beyond that, reach for a component instead.

### Component Communication — No Direct Coupling
Components should not `#include` each other's headers or hold raw pointers to sibling components. Use the owning actor as the mediator, or use delegates:

```cpp
// WRONG — HealthComponent directly knows about UIComponent
void UHealthComponent::OnDamaged(float Amount)
{
    GetOwner()->FindComponentByClass<UUIComponent>()->UpdateHealthBar(CurrentHealth);
}

// CORRECT — HealthComponent broadcasts; UIComponent listens
// In UHealthComponent:
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnHealthChanged, float, NewHealth, float, MaxHealth);
UPROPERTY(BlueprintAssignable)
FOnHealthChanged OnHealthChanged;

// In AMyCharacter::BeginPlay:
HealthComponent->OnHealthChanged.AddDynamic(UIComponent, &UUIComponent::UpdateHealthBar);
```

---

## UInterface — Usage Rules

### Declaration Pattern
```cpp
// MyDamageable.h
UINTERFACE(MinimalAPI, Blueprintable)
class UMyDamageable : public UInterface
{
    GENERATED_BODY()
};

class IMyDamageable
{
    GENERATED_BODY()
public:
    // BlueprintNativeEvent — C++ default impl, Blueprint can override
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Damage")
    void TakeDamage(float Amount);
    virtual void TakeDamage_Implementation(float Amount) {}

    // BlueprintImplementableEvent — Blueprint must implement; C++ cannot call directly
    UFUNCTION(BlueprintImplementableEvent, BlueprintCallable)
    void OnDied();
};
```

### Calling Rules
```cpp
// Always use Execute_ — works for C++ and Blueprint implementors
IMyDamageable::Execute_TakeDamage(TargetActor, 25.f);

// Always check Implements<> first
if (TargetActor->Implements<UMyDamageable>())
{
    IMyDamageable::Execute_TakeDamage(TargetActor, 25.f);
}

// WRONG — direct vtable call crashes on Blueprint implementors
Cast<IMyDamageable>(TargetActor)->TakeDamage(25.f);
```

### When To Use UInterface vs Base Class
- Use **UInterface** when unrelated class hierarchies share a behavior (an `AActor`, a `UActorComponent`, and a `UObject` all being "damageable")
- Use **base class inheritance** when all implementors share a common `UObject`/`AActor` ancestor and the relationship is a true "is-a"

---

## Delegate Type Selection

Choosing the wrong delegate type costs either performance (dynamic on hot paths) or functionality (multicast in Blueprint-accessible API).

| Type | Macro | Blueprint | Serializable | Overhead | Use For |
|---|---|---|---|---|---|
| Multicast (C++ only) | `DECLARE_MULTICAST_DELEGATE` | No | No | Lowest | Internal system events not exposed to designers |
| Dynamic Multicast | `DECLARE_DYNAMIC_MULTICAST_DELEGATE` | Yes | Yes | Medium | Events designers bind to in Blueprint |
| Sparse Dynamic | `DECLARE_SPARSE_DYNAMIC_MULTICAST_DELEGATE` | Yes | Yes | Zero when unused | Events on frequently-spawned actors (projectiles, pickups) |

```cpp
// Internal system — no Blueprint needed, use multicast
DECLARE_MULTICAST_DELEGATE_OneParam(FOnTileReady, URunnerTileBase*);
FOnTileReady OnTileReady; // zero Blueprint overhead

// Designer-facing event — use dynamic multicast
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPlayerDied, APlayerState*, DeadPlayer);
UPROPERTY(BlueprintAssignable, Category="Events")
FOnPlayerDied OnPlayerDied;

// Frequently-spawned actor (e.g., AProjectile) — use sparse to avoid per-instance memory cost
DECLARE_SPARSE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnProjectileHit, AProjectile, ..., FHitResult, Hit);
UPROPERTY(BlueprintAssignable)
FOnProjectileHit OnProjectileHit;
```

---

## Subsystem as Singleton Replacement

### The Rule
Never implement a singleton (static instance, `Get()` class method) in UE5. Use `UGameInstanceSubsystem` instead.

```cpp
// WRONG — static singleton; crashes on PIE restart, leaks between sessions
class UMyManager
{
public:
    static UMyManager* Get() { return Instance; }
private:
    static UMyManager* Instance;
};

// CORRECT — engine-managed, GC-safe, accessible globally
UCLASS()
class UMyManagerSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()
public:
    // Access from anywhere with a world or game instance pointer:
    // GameInstance->GetSubsystem<UMyManagerSubsystem>()
    // UGameplayStatics::GetGameInstance(World)->GetSubsystem<UMyManagerSubsystem>()
};
```

### Which Subsystem Type

| Type | Lifetime | Use For |
|---|---|---|
| `UGameInstanceSubsystem` | Entire game session (survives level loads) | Save data, matchmaking, analytics, audio state |
| `UWorldSubsystem` | One world/level | Tile manager, difficulty ramp, per-level gameplay state |
| `ULocalPlayerSubsystem` | Per local player | Per-player UI state, input configuration |
| `UEngineSubsystem` | Entire engine lifetime | Editor tools, low-level services |

Gate subsystem creation with `ShouldCreateSubsystem()` to exclude subsystems from contexts where they are not needed (e.g., dedicated server, commandlets).

---

## Object Pooling — Rule for Hot Paths

`SpawnActor` / `DestroyActor` in a hot path (per-shot, per-frame, per-obstacle spawn) causes GC pressure, hitches, and fragmented memory. **Always pool any actor that spawns and destroys frequently.**

Pool pattern requirements:
1. On acquire: call `SetActorHiddenInGame(false)`, `SetActorEnableCollision(true)`, `SetActorTickEnabled(true)`
2. On release: call `SetActorHiddenInGame(true)`, `SetActorEnableCollision(false)`, `SetActorTickEnabled(false)`, reset all state
3. The pool's raw `T*` array is safe only if pooled actors are also registered with the world (`UWorld::Actors` holds a GC-tracked reference) — never call `DestroyActor` on a pooled actor from outside the pool

See `refs/systems-engineer/crash-prevention.md` CRASH-003 for the GC analysis of raw pointer safety in non-UObject pool containers.
