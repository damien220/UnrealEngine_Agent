# C++ Casting & Type Safety — UE5 Reference

## Cast Function Comparison

| Function | On Failure | Build Config | Use When |
|---|---|---|---|
| `Cast<T>(Obj)` | Returns `nullptr` | All | You're not sure the type matches |
| `CastChecked<T>(Obj)` | Asserts in Debug; UB in Shipping | Debug asserts, Shipping silent crash | You are certain the type matches (invariant) |
| `static_cast<T*>(Obj)` | No check at all | All | **Never on UObjects** |
| `IsA<T>()` | Returns `bool` | All | Type check without needing the cast result |
| `Implements<UInterface>()` | Returns `bool` | All | Checking interface implementation before calling |

---

## Rules

### `Cast<T>` — The Default Choice

```cpp
// Always check the result — Cast returns nullptr when the type doesn't match
if (AMyEnemy* Enemy = Cast<AMyEnemy>(OtherActor))
{
    Enemy->TakeDamage(10.f, ...);
}
// If OtherActor is not an AMyEnemy, Enemy is nullptr — no crash, no assert
```

Use `Cast<T>` for any cast where the type match is not guaranteed — player controller casts, hit result actors, delegate payloads.

### `CastChecked<T>` — Only For True Invariants

```cpp
// CORRECT use: GameMode is always AMyGameMode in this project — it's an engine invariant
AMyGameMode* GM = CastChecked<AMyGameMode>(GetWorld()->GetAuthGameMode());

// WRONG use: Actor could be anything — CastChecked crashes in Shipping if wrong
AMyEnemy* Enemy = CastChecked<AMyEnemy>(HitResult.GetActor()); // never do this
```

In Shipping builds `CastChecked` does not assert — it accesses invalid memory silently. Treat it as a contract: "this cast failing is a programming error that should never reach players."

### `static_cast` — Forbidden on UObjects

```cpp
// NEVER — bypasses UE's reflection, breaks GC tracking, causes type confusion
AMyActor* A = static_cast<AMyActor*>(SomeUObjectPtr);

// ALWAYS use Cast<> instead
AMyActor* A = Cast<AMyActor>(SomeUObjectPtr);
```

`static_cast` does not interact with UE's `UObject` type hierarchy or `UCLASS` reflection. It will produce incorrect results on `UObject` multiple inheritance (e.g., when an actor implements a `UInterface`) and is undefined behavior if the types are incompatible.

---

## Interface Type Checks

Casting to an interface requires a different call path than casting to a class.

```cpp
// Check first — never assume an actor implements an interface
if (OtherActor->Implements<UMyDamageable>())
{
    // Call via Execute_ — works for both C++ and Blueprint implementors
    IMyDamageable::Execute_TakeDamage(OtherActor, 10.f);
}

// WRONG — direct interface pointer dereference crashes if implemented in Blueprint
IMyDamageable* Damageable = Cast<IMyDamageable>(OtherActor);
Damageable->TakeDamage(10.f); // crashes on Blueprint implementors
```

**Why `Execute_` is mandatory:** Blueprint-implemented interfaces are not C++ vtable calls — the engine dispatches them through the reflection system. Calling the function directly on the interface pointer skips this dispatch and accesses invalid memory on Blueprint actors.

### `TScriptInterface` — For Storing Interface References

```cpp
// Storing an interface reference that Blueprint can also hold
UPROPERTY(BlueprintReadWrite)
TScriptInterface<IMyDamageable> DamageTarget;

// Setting it
DamageTarget = TScriptInterface<IMyDamageable>(SomeActor);

// Calling it — still use Execute_
if (DamageTarget)
{
    IMyDamageable::Execute_TakeDamage(DamageTarget.GetObject(), 10.f);
}
```

`TScriptInterface<T>` stores both the `UObject*` (for GC tracking) and the interface pointer (for C++ dispatch) — it is the only correct way to hold an interface reference in a `UPROPERTY`.

---

## `IsA<T>` — Type Check Without Casting

```cpp
// When you only need to know the type, not use the cast result
if (OtherActor->IsA<AMyEnemy>())
{
    OverlapCount++;
    // No cast needed — just counting enemies
}

// IsA also works with class objects
if (OtherActor->IsA(EnemyClass))  // EnemyClass is a TSubclassOf<AActor>
{
    ...
}
```

Prefer `IsA<T>()` over `Cast<T>() != nullptr` when you don't use the cast result — it signals intent clearly and avoids an unused local variable.
