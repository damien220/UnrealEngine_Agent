# C++/Blueprint Architecture Boundary — UE5 Reference

## The Core Rule: Tick Belongs in C++

Blueprint VM executes via interpreted bytecode with per-node overhead, cache misses on the Blueprint object graph, and no compiler optimization. At tick frequency this overhead is multiplied by every active actor.

```cpp
// WRONG — Blueprint Tick Event at 60Hz for gameplay logic
// Every node in the Blueprint graph is a virtual dispatch. At 200 enemies this destroys frame budget.

// CORRECT — C++ Tick with controlled rate
AMyEnemy::AMyEnemy()
{
    PrimaryActorTick.bCanEverTick = true;
    PrimaryActorTick.TickInterval = 0.05f; // 20Hz — deliberate, not default
}

void AMyEnemy::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    UpdateMovementPrediction(DeltaTime); // C++ only — no Blueprint node calls here
}
```

**Rule**: Any function called every frame — including Tick, collision overlap updates, animation state evaluation — must live in C++. Blueprint Tick is only acceptable on UI widgets with a single node and no iteration.

---

## Data Types That Require C++

Blueprint's type system is a subset of C++. The following types have no Blueprint equivalent and must be declared in C++:

| C++ Type | Blueprint Equivalent | Decision |
|---|---|---|
| `uint16`, `int8`, `int64` | None | Implement in C++ |
| `TMultiMap<K, V>` | None | Implement in C++ |
| `TSet<T>` with custom hash | None | Implement in C++ |
| `FName` arrays as keys | None | Implement in C++ |
| Bitfields (`uint8 bFlag : 1`) | None | Implement in C++ |
| `TOptional<T>` | None | Implement in C++ |

Exposing aggregated results (e.g., a `USTRUCT` wrapping a query result) is fine — just implement the underlying logic in C++.

---

## Engine Extensions That Always Require C++

These systems hook into engine internals that Blueprint cannot access:

- **Custom Character Movement** — `UCharacterMovementComponent` overrides require overriding virtual C++ functions. Blueprint CMC subclasses cannot override movement math.
- **Physics Callbacks** — `OnComponentHit`, `OnComponentBeginOverlap`, and Chaos callbacks must be bound with `AddDynamic` in C++; Blueprint-only actors miss collision events under network load.
- **Custom Collision Channels** — defined in `DefaultEngine.ini` and referenced by enum in C++; Blueprint can read them but not define them.
- **Input Processing** — `IInputProcessor` for raw pre-actor-stack input must be C++. Enhanced Input mappings can live in Blueprint but complex input routing requires C++.
- **Async/Latent Nodes** — custom latent Blueprint nodes (`UFUNCTION(BlueprintCallable, meta=(Latent, ...))`) are declared and implemented in C++.
- **Net serialization** — custom `NetSerialize` on `USTRUCT` is C++ only. Blueprint structs use default replication which cannot be customized.

---

## UFUNCTION Exposure Macros

C++ systems are exposed to Blueprint via three distinct macros, each with a different ownership model:

### `BlueprintCallable` — C++ implements, Blueprint calls
```cpp
// C++ owns the implementation. Blueprint can call it but not override it.
UFUNCTION(BlueprintCallable, Category = "Combat")
void ApplyDamage(float Amount, AActor* Instigator);
```

### `BlueprintImplementableEvent` — C++ declares, Blueprint implements
```cpp
// C++ fires the event. Blueprint provides the response. C++ has no default body.
// Use for designer hooks: "on ability activated", "on death", "on level up".
UFUNCTION(BlueprintImplementableEvent, Category = "Combat")
void OnDamageTaken(float Amount);
// C++ calls: OnDamageTaken(50.f);  — Blueprint sees it as an event node
```

### `BlueprintNativeEvent` — C++ provides default, Blueprint can override
```cpp
// C++ declares and provides _Implementation(). Blueprint can override the behavior.
// Use when you need a sensible default but designers may want to customize.
UFUNCTION(BlueprintNativeEvent, Category = "Combat")
bool CanActivateAbility();
virtual bool CanActivateAbility_Implementation();  // C++ body here
```

**Choosing between them:**

| Macro | C++ owns logic? | Blueprint can override? | Use case |
|---|---|---|---|
| `BlueprintCallable` | Yes | No | Utility functions, system calls |
| `BlueprintImplementableEvent` | No (event only) | Yes (required) | Designer hooks, callbacks |
| `BlueprintNativeEvent` | Yes (default) | Yes (optional) | Overrideable systems with fallback |

---

## What Belongs in Blueprint

Blueprint is the designer-facing API layer, not the implementation layer. Appropriate Blueprint content:

- **High-level game flow** — level sequencing, game state transitions, win/lose conditions that are data-driven
- **UI logic** — Widget Blueprints with event responses and data binding via C++ exposed properties
- **Prototyping** — initial feature discovery before performance requirements are known
- **Sequencer-driven events** — cinematic events, cutscene triggers, environment storytelling
- **Data configuration** — setting `EditDefaultsOnly` properties on ability, character, and item Data Assets
- **Designer-authored responses** — Blueprint implementations of `BlueprintImplementableEvent` hooks

**Anti-patterns to reject in Blueprint:**
- Any loop over an array of actors every frame
- Casting chains (`Cast → Cast → Cast`) to access nested data — expose a single `BlueprintCallable` instead
- Blueprint-side math-heavy AI evaluation — move to C++ BehaviorTree tasks
- Timer-based Blueprint sequences for gameplay (use C++ timer manager or Gameplay Abilities)

---

## Blueprint Function Libraries

For utility functions that designers call frequently across many Blueprints, use a `UBlueprintFunctionLibrary`:

```cpp
UCLASS()
class MYGAME_API UMyGameplayLibrary : public UBlueprintFunctionLibrary
{
    GENERATED_BODY()

public:
    // Static — no instance required. Accessible from any Blueprint.
    UFUNCTION(BlueprintCallable, Category = "Gameplay|Utility")
    static float CalculateDamageWithResistance(float RawDamage, float ResistancePercent);

    UFUNCTION(BlueprintPure, Category = "Gameplay|Utility")
    static bool IsActorInTeam(AActor* Actor, ETeamID Team);
};
```

`BlueprintPure` functions have no execution pin — use for getters and computations with no side effects.
