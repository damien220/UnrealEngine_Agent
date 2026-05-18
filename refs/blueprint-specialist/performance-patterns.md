# Blueprint Performance Patterns — UE5 Reference

## The Blueprint VM Cost Model

Blueprint executes as interpreted bytecode — each node is a virtual dispatch with dynamic type lookup. A C++ equivalent of Blueprint Tick typically runs 10–20× faster at equivalent logic complexity. This is not a reason to avoid Blueprint; it is a constraint on which logic belongs in Blueprint.

**Rule:** Any logic that runs more frequently than once per second per actor, or that iterates a collection larger than ~10 elements, belongs in C++.

---

## Tick Discipline

### Never enable Tick unless you have measured its necessity

```
Default behavior: Blueprint classes inherit Tick enabled from their C++ parent.
Every spawned instance ticks every frame.
100 enemies with Blueprint Tick = 100 VM dispatches/frame before any logic runs.
```

**Disable Tick in the Blueprint Class Settings** if the class does not need it. Re-enable only with explicit justification.

### Alternatives to Tick

| Goal | Tick approach (avoid) | Better approach |
|---|---|---|
| Animate a value over time | `Interp To` in Tick | Timeline node or `UWidgetAnimation` |
| Check a condition periodically | Branch in Tick every frame | `SetTimer by Function Name` at desired interval |
| Detect proximity | Distance check in Tick | `SphereComponent` overlap events |
| Smooth HUD value | Lerp in Tick | `FInterpTo` in C++ or Timeline |
| Follow a spline | Update position in Tick | `InterpToMovement` component |

### Timer-Based Polling Pattern

```
[Event BeginPlay]
  → [Set Timer by Function Name]
    Function Name: "CheckProximity"
    Time: 0.2  ← 5Hz instead of 60Hz
    Looping: True

[Function: CheckProximity]
  → [your condition logic here]
```

5Hz polling for AI perception is indistinguishable from 60Hz to players — and costs 12× less.

---

## GetAllActorsOfClass — Never in Hot Paths

`GetAllActorsOfClass` and `GetAllActorsWithInterface` iterate every actor in the world every call. At 200 actors and 60Hz:

```
200 actors × 60 calls/sec = 12,000 actor iterations/sec from one Blueprint node
```

**Correct patterns:**

| When to call | Pattern |
|---|---|
| BeginPlay (once) | Cache results in an array variable; refresh on actor spawn/death |
| On a low-frequency timer (0.5–1Hz) | Acceptable if the world is small (< 50 actors) |
| In Tick | Never |
| On input events | Never |

```
BAD — in Tick:
[Event Tick] → [GetAllActorsOfClass: AEnemy] → [ForEach] → [distance check]

GOOD — cached at BeginPlay, updated on spawn:
[Event BeginPlay] → [GetAllActorsOfClass: AEnemy] → [Set AllEnemies (Array variable)]

In GameMode/Subsystem — register enemies on spawn:
[On Enemy Spawned] → [AllEnemies.Add(NewEnemy)]
[On Enemy Died]   → [AllEnemies.Remove(DeadEnemy)]
```

---

## ForEach Loop Optimization

Blueprint `ForEach` with Break compiles to a loop with a branch on every iteration. For collections > 50 elements on a hot path, move the loop to C++:

```cpp
// C++ equivalent — no branch overhead per iteration
UFUNCTION(BlueprintCallable, Category = "Enemies")
AActor* FindNearestEnemy(const TArray<AActor*>& Enemies, FVector Origin)
{
    AActor* Nearest = nullptr;
    float NearestDistSq = TNumericLimits<float>::Max();
    for (AActor* Enemy : Enemies)
    {
        if (!IsValid(Enemy)) continue;
        float DistSq = FVector::DistSquared(Origin, Enemy->GetActorLocation());
        if (DistSq < NearestDistSq)
        {
            NearestDistSq = DistSq;
            Nearest = Enemy;
        }
    }
    return Nearest;
}
```

Expose via `BlueprintCallable` — Blueprint calls it with one node.

---

## Struct Input Pins — Batch Data Passing

Passing many values as individual pins creates one wire evaluation per value. Wrapping them in a struct creates one evaluation for the bundle:

```
BAD — 5 pins evaluated separately:
[Function: SpawnEnemy(Location, Rotation, Health, Team, Level)]
  5 parameter reads on every call

GOOD — 1 struct pin:
USTRUCT(BlueprintType)
struct FEnemySpawnParams { Location, Rotation, Health, Team, Level };

[Function: SpawnEnemy(SpawnParams: FEnemySpawnParams)]
  1 struct read, values accessed by the function internally
```

Structs also make Blueprint function signatures stable — adding a field does not break existing Blueprint call sites (the new field gets a default value).

---

## Cast Cost

`Cast To` in Blueprint is a `dynamic_cast` under the hood — O(class hierarchy depth). A chain of casts:

```
[Cast to AEnemyA] → on fail → [Cast to AEnemyB] → on fail → [Cast to AEnemyC]
```

...is three dynamic_cast calls per execution. At 100 actors per frame, 3 casts each = 300 dynamic_cast calls/frame from one subgraph. Replace with an Interface call — one virtual dispatch regardless of class hierarchy depth.

---

## Blueprint Profiler Usage

`Window → Developer Tools → Blueprint Profiler`

Key columns:
- **Inclusive Time**: total time including called sub-functions — sort by this first
- **Exclusive Time**: time in this node only, not sub-calls — identifies hot nodes within a function
- **Total Time**: summed across all invocations in the profiling window

Thresholds:
| Value | Action |
|---|---|
| Function Inclusive Time > 0.1ms/call | Investigate — likely a loop or expensive query |
| Total Frame Time from BP > 2ms | C++ migration needed for the heaviest functions |
| Single node Exclusive Time > 0.05ms | Usually a `GetAllActors` or an unoptimized query |

```
stat blueprints          — real-time per-class Blueprint execution cost
stat game                — total game thread time (Blueprint contributes to this)
```

---

## Blueprint Size Audit

Large Blueprints (many Functions, large variable counts) have measurable load and compile cost. Check in the editor:

```
File → Reference Viewer         — find all assets this Blueprint hard-references (forces loading)
Window → Size Map               — visualize memory cost of all referenced assets
Developer → Asset Audit         — find unexpectedly large or multiply-loaded assets
```

Every hard asset reference (`TObjectPtr`, direct asset path in a property) forces that asset to load when this Blueprint loads. Use `TSoftObjectPtr` for assets that should load on-demand.
