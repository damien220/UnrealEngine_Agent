# Object Lifecycle & Creation — UE5 Reference

## Initialization Phase Order

UE5 constructs objects through a fixed sequence of phases. Using the wrong phase for a given operation causes crashes, CDO corruption, or silent failures.

| Phase | When It Fires | What Is Safe | What Is Forbidden |
|---|---|---|---|
| `Constructor` | CDO creation at engine startup | `CreateDefaultSubobject`, setting `UPROPERTY` defaults | `GetWorld()`, `AddDynamic`, `SpawnActor`, `NewObject` on external outers |
| `PostInitializeComponents` | After all components are created and initialized | Cross-wiring component references, component configuration that needs other components | Gameplay state changes, subsystem access |
| `BeginPlay` | World is valid and ticking has started | Subsystem access, delegate binding, timer setup, `GetWorld()`, `SpawnActor` | `CreateDefaultSubobject` |
| `PostLoad` | After asset deserialization from disk | Fixing up asset references, migration fixups | Assuming world exists — `GetWorld()` returns null here |

### CDO Trap — The Most Common Constructor Crash

Every `UCLASS` has a Class Default Object (CDO) constructed at engine startup **without a world**. Code in the constructor runs on the CDO first, then again on each spawned instance. The consequence:

```cpp
// WRONG — GetWorld() is null on CDO; crashes or silently skips
AMyActor::AMyActor()
{
    UWorld* World = GetWorld();           // null on CDO
    World->SpawnActor<AMyHelper>(...);    // null dereference → crash
}

// WRONG — AddDynamic in constructor binds on CDO, causes duplicate bindings on instances
AMyActor::AMyActor()
{
    MyComponent->OnComponentHit.AddDynamic(this, &AMyActor::OnHit); // double-bind
}

// CORRECT — move world-dependent work to BeginPlay
void AMyActor::BeginPlay()
{
    Super::BeginPlay();
    GetWorld()->SpawnActor<AMyHelper>(...);
    MyComponent->OnComponentHit.AddDynamic(this, &AMyActor::OnHit);
}
```

Detect CDO context when you must: `if (HasAnyFlags(RF_ClassDefaultObject)) return;`

---

## Object Creation Rules

Three creation functions exist for different contexts. Using the wrong one crashes or leaks.

### `CreateDefaultSubobject<T>` — Constructor Only

```cpp
// CORRECT — called in constructor, registers component with the class
AMyCharacter::AMyCharacter()
{
    HealthComponent = CreateDefaultSubobject<UHealthComponent>(TEXT("HealthComponent"));
    HealthComponent->SetupAttachment(RootComponent);
}

// WRONG — called at runtime, asserts and crashes
void AMyCharacter::BeginPlay()
{
    // CreateDefaultSubobject is constructor-only — this asserts
    SomeComp = CreateDefaultSubobject<UMyComponent>(TEXT("Late"));
}
```

### `NewObject<T>(Outer)` — Runtime UObject Creation

```cpp
// Correct: outer is this actor — component lives as long as the actor
UMyDataComponent* Data = NewObject<UMyDataComponent>(this, UMyDataComponent::StaticClass());
Data->RegisterComponent(); // required if it's an ActorComponent

// WRONG: transient outer or null outer — GC collects it next cycle
UMyDataComponent* Data = NewObject<UMyDataComponent>(); // outer = GetTransientPackage()
// Data is gone after next GC pass unless also stored in a UPROPERTY
```

**Rule:** The outer controls GC lifetime. If the outer is destroyed, the created object is eligible for collection. Always store the result in a `UPROPERTY` member or pass a stable outer.

### `SpawnActor` vs `SpawnActorDeferred` — For Actors That Need Pre-BeginPlay Setup

```cpp
// SpawnActor — BeginPlay fires immediately; too late to configure the actor
AMyProjectile* P = World->SpawnActor<AMyProjectile>(ProjectileClass, Transform);
// P->BeginPlay() has already fired — setting InitialDamage here is a race

// SpawnActorDeferred — configure before BeginPlay
AMyProjectile* P = World->SpawnActorDeferred<AMyProjectile>(
    ProjectileClass, Transform, Instigator);
P->InitialDamage = ComputedDamage;   // safe — BeginPlay not yet called
UGameplayStatics::FinishSpawningActor(P, Transform); // triggers BeginPlay
```

Use `SpawnActorDeferred` + `FinishSpawningActor` whenever spawned actors need initialization data that BeginPlay must see.

---

## Component Registration

Components created at runtime with `NewObject` must be manually registered; `CreateDefaultSubobject` does this automatically.

```cpp
UMyComponent* Comp = NewObject<UMyComponent>(this);
Comp->RegisterComponent();          // adds to world tick, enables overlap events
AddInstanceComponent(Comp);         // makes it show in editor Details panel (optional)
```

Failing to call `RegisterComponent` on a runtime component produces a component that exists in memory but is invisible to the engine — no ticking, no physics, no overlap events.
