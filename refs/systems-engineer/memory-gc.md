# Memory Management & Garbage Collection — UE5 Reference

## How UE5 Garbage Collection Works

UE5's GC is a mark-and-sweep collector that runs on a background thread (reachability analysis) with a game-thread stop-the-world phase (unreachable object destruction). It does **not** use reference counting for `UObject`-derived types.

The only way to tell the GC that a `UObject*` is reachable is via `UPROPERTY()`. A raw `UObject*` without `UPROPERTY` is invisible to the GC — the pointed-to object can be collected at any time, leaving a dangling pointer that crashes on the next access.

---

## UPROPERTY — The Mandatory GC Root

**Rule**: Every `UObject`-derived pointer stored as a class member **must** be declared with `UPROPERTY()`.

```cpp
// WRONG — GC does not see this pointer. The object will be collected.
class AMyActor : public AActor
{
    UMyComponent* MyComp;        // No UPROPERTY — dangling pointer after GC
    UTexture2D* CachedTexture;   // Same issue — will crash on access
};

// CORRECT — UPROPERTY marks these as GC roots
class AMyActor : public AActor
{
    UPROPERTY()
    UMyComponent* MyComp;

    UPROPERTY()
    UTexture2D* CachedTexture;

    // With editor/Blueprint exposure
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Config")
    UStaticMesh* WeaponMesh;
};
```

**UPROPERTY specifiers do not affect GC root behavior** — even a bare `UPROPERTY()` with no specifiers is sufficient to prevent collection. Specifiers only affect editor exposure and replication.

### Local Variables and Function Parameters
Local `UObject*` variables on the stack do not need `UPROPERTY` — they are automatically rooted for the duration of the function call. Only class members require it.

---

## TWeakObjectPtr — Non-Owning References

Use `TWeakObjectPtr<T>` when you need a reference to a `UObject` but your class does not own it. Weak pointers do not prevent GC collection.

```cpp
// Scenario: caching a reference to another actor you don't own
UPROPERTY()  // Note: TWeakObjectPtr also needs UPROPERTY to be serialized/replicated
TWeakObjectPtr<APlayerController> CachedController;

void AMyActor::SetController(APlayerController* PC)
{
    CachedController = PC;
}

void AMyActor::UseController()
{
    // WRONG — split check and use; GC can run between the two lines
    if (CachedController.IsValid())
    {
        CachedController->ClientPlayForceFeedback(...);  // Unsafe — GC could fire here
    }

    // CORRECT — resolve to local, check local
    APlayerController* PC = CachedController.Get();
    if (IsValid(PC))
    {
        PC->ClientPlayForceFeedback(...);  // PC is a stack reference — safe for this call
    }
}
```

**When to use `TWeakObjectPtr` vs raw `UPROPERTY()`**:
- Use `TWeakObjectPtr` when the referenced object has an independent lifetime and may be destroyed before the owner.
- Use raw `UPROPERTY()` (owning reference) when the member should prevent the referenced object from being GC'd.

---

## TSharedPtr and TWeakPtr — Non-UObject Heap Allocations

For heap-allocated objects that do **not** derive from `UObject`, use `TSharedPtr<T>` / `TWeakPtr<T>`. These are reference-counted, not GC-tracked.

```cpp
// Correct pattern for non-UObject data structures
TSharedPtr<FMyPathfindingData> PathCache;
TSharedPtr<FMyInventoryGrid> InventoryGrid;

// Creating
PathCache = MakeShared<FMyPathfindingData>();

// Non-owning reference to shared data
TWeakPtr<FMyPathfindingData> WeakPath = PathCache;

void AMyActor::UsePath()
{
    TSharedPtr<FMyPathfindingData> PinnedPath = WeakPath.Pin();
    if (PinnedPath.IsValid())
    {
        PinnedPath->Recalculate();
    }
}
```

**Never mix `TSharedPtr` and `UObject`**: `TSharedPtr<UObject>` bypasses GC tracking and causes double-free crashes. Use `UPROPERTY()` + `TWeakObjectPtr` for all `UObject` types.

---

## Raw AActor* Pointers Across Frames

Actors can be destroyed mid-frame via `Destroy()`, level streaming, or network relevancy changes. A raw `AActor*` that was valid when stored may be invalid one frame later.

```cpp
// WRONG — storing raw AActor* for use next frame
AActor* Target;  // No UPROPERTY — GC risk
void AMyActor::SetTarget(AActor* NewTarget) { Target = NewTarget; }

// CORRECT — UPROPERTY + IsValid check at every access point
UPROPERTY()
AActor* Target;

void AMyActor::AttackTarget()
{
    if (!IsValid(Target)) return;  // Null AND pending-kill check
    // ... safe to use Target here
}
```

For actors you don't own, prefer `TWeakObjectPtr<AActor>` over `UPROPERTY() AActor*` to make the non-owning intent explicit.

---

## IsValid() vs != nullptr

**Rule**: Always call `IsValid()`, never `!= nullptr`, when checking `UObject` validity.

| Check | Catches null? | Catches pending-kill? | Use case |
|---|---|---|---|
| `!= nullptr` | Yes | **No** | Raw C++ pointers (non-UObject) only |
| `IsValid(Ptr)` | Yes | **Yes** | All UObject-derived pointers |
| `Ptr.IsValid()` (TWeakObjectPtr) | Yes | Yes | Weak pointer before pinning |

```cpp
// WRONG — object may be non-null but pending destruction
if (MyComponent != nullptr)
{
    MyComponent->DoSomething(); // Crash if pending kill
}

// CORRECT
if (IsValid(MyComponent))
{
    MyComponent->DoSomething(); // Safe — null and pending-kill both caught
}

// For TWeakObjectPtr, resolve first then IsValid on the raw pointer
UMyComponent* Comp = WeakComp.Get();
if (IsValid(Comp))
{
    Comp->DoSomething();
}
```

---

## TObjectPtr — UPROPERTY-Only Member Type

`TObjectPtr<T>` is a typed pointer wrapper introduced in UE5 for use exclusively in `UPROPERTY` fields. It enables lazy serialization and cook-time tracking, and is the preferred form for `UPROPERTY` member declarations in UE5.

```cpp
// UE5 preferred style for UPROPERTY members
UPROPERTY(EditDefaultsOnly)
TObjectPtr<UStaticMesh> WeaponMesh;

UPROPERTY()
TObjectPtr<UMyComponent> CachedComp;

// Local variables — use raw pointer, NOT TObjectPtr
void AMyActor::DoWork()
{
    UMyComponent* LocalComp = GetComponentByClass<UMyComponent>();  // raw pointer — correct
    // TObjectPtr<UMyComponent> LocalComp = ...;  // WRONG — TObjectPtr is for members only
}
```

---

## GC Timing and Safe Patterns

GC reachability analysis runs on a background thread; object destruction runs on the game thread between frames. The practical consequences:

1. **Never access UObjects from background threads** — GC may be destroying them concurrently.
2. **After `Destroy()` is called, the actor is pending-kill immediately** — `IsValid()` returns false from that point, even if the object hasn't been collected yet.
3. **Delegate-bound UObject callbacks must be unbound before the object is destroyed** — use `RemoveDynamic` in `EndPlay`.

```cpp
void AMyActor::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    // Unbind all delegates to prevent callbacks into a destroyed object
    if (IsValid(SomeOtherActor))
    {
        SomeOtherActor->OnSomeEvent.RemoveDynamic(this, &AMyActor::HandleEvent);
    }

    // Clear timer handles to prevent timer callbacks post-destruction
    GetWorldTimerManager().ClearTimer(MyTimerHandle);

    Super::EndPlay(EndPlayReason);
}
```
