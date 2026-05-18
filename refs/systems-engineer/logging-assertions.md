# Logging & Assertions — UE5 Reference

## Log Category Definition

Every module must define its own log category. Using `LogTemp` in shipped code makes logs unsearchable, unfilterable, and indistinguishable from other systems' noise.

```cpp
// MyModule.h — declare the category (visible to other files that include this header)
DECLARE_LOG_CATEGORY_EXTERN(LogMyModule, Log, All);

// MyModule.cpp — define it (one definition per category, in the .cpp)
DEFINE_LOG_CATEGORY(LogMyModule);

// Usage anywhere in the module
UE_LOG(LogMyModule, Warning, TEXT("Health dropped to %.1f"), CurrentHealth);
UE_LOG(LogMyModule, Error, TEXT("Null subsystem in %s"), *GetName());
```

### Log Verbosity Levels

| Level | Use For | Shipping Build |
|---|---|---|
| `Fatal` | Unrecoverable state — engine will crash after logging | Included |
| `Error` | Definite bug or data integrity failure | Included |
| `Warning` | Unexpected but recoverable state | Included |
| `Log` | Normal informational events | Stripped by default |
| `Verbose` | Detailed trace for debugging a specific system | Always stripped |
| `VeryVerbose` | Per-frame trace | Always stripped |

Set default verbosity per category in `DefaultEngine.ini`:
```ini
[Core.Log]
LogMyModule=Verbose   ; Enable verbose for this category in non-Shipping builds
```

---

## Assertion Semantics

Choosing the wrong assertion causes either missed bugs (too lenient) or production crashes (too strict).

| Macro | Behavior in Debug | Behavior in Shipping | Use When |
|---|---|---|---|
| `check(expr)` | Asserts + crashes | Crashes (or skips with some configs) | Invariant that can never fail; programmer error if it does |
| `checkf(expr, fmt, ...)` | Asserts with message + crashes | Same | Same as check, with diagnostic message |
| `ensure(expr)` | Fires once, logs callstack, continues | Skipped entirely | "This shouldn't happen but we can recover"; catches bugs without killing the game |
| `ensureMsgf(expr, fmt, ...)` | Same as ensure + formatted message | Skipped | Preferred over bare `ensure` — always add context |
| `verify(expr)` | Same as check | **Expression still evaluated** | When the expression has side effects that must run in Shipping |

```cpp
// check — use for programming invariants
void UMyComponent::ActivateAbility(UGameplayAbility* Ability)
{
    check(Ability); // If this is null, it's a caller bug — crash to catch it in development
    Ability->Activate();
}

// ensure — use for unexpected-but-survivable states
void AMyCharacter::OnDamaged(float Amount)
{
    // This shouldn't be negative, but we can clamp and continue
    ensureMsgf(Amount >= 0.f, TEXT("Negative damage applied: %.1f from %s"),
        Amount, *GetName());
    Amount = FMath::Max(0.f, Amount);
    Health -= Amount;
}

// verify — when the expression must run in Shipping
bool bSuccess = verify(TryConnectToServer()); // TryConnectToServer() runs in all configs
```

### The `ensure` One-Shot Rule

`ensure` fires **once per unique callsite per session** in non-Shipping builds, then is silently suppressed. This is intentional — it catches the bug once without flooding the log. Do not rely on `ensure` to catch every occurrence of a condition; use `check` if every occurrence matters, or `UE_LOG` with `Error` if you need to see all occurrences.

---

## Structured Log Patterns

```cpp
// Contextual log — always include object name and relevant values
UE_LOG(LogMyModule, Warning,
    TEXT("[%s] SpawnTile failed: TileClass is null. ActiveTiles=%d"),
    *GetName(), ActiveTiles.Num());

// Conditional verbose log — zero cost in Shipping
UE_LOG(LogMyModule, Verbose, TEXT("Pool acquire: class=%s available=%d"),
    *PooledClass->GetName(), Available.Num());

// Log + ensure together — catches the bug and provides recovery log
if (!ensureMsgf(IsValid(Subsystem), TEXT("MySubsystem is null in %s"), *GetName()))
{
    UE_LOG(LogMyModule, Error, TEXT("Skipping operation — subsystem unavailable"));
    return;
}
```

---

## Debug Console Variables (CVars)

For systems that need runtime-toggleable debug visualization or logging:

```cpp
// Declare — typically at file scope in the .cpp
static TAutoConsoleVariable<bool> CVarMySystemDebug(
    TEXT("MySystem.Debug"),
    false,
    TEXT("Enable debug drawing for MySystem"),
    ECVF_Cheat
);

// Use — gated so zero cost when off
void AMyActor::Tick(float DeltaTime)
{
    #if ENABLE_DRAW_DEBUG
    if (CVarMySystemDebug.GetValueOnGameThread())
    {
        DrawDebugBox(GetWorld(), GetActorLocation(), FVector(50.f), FColor::Green, false, -1.f);
    }
    #endif
}
```

Gate all `DrawDebug*` calls inside `#if ENABLE_DRAW_DEBUG` — debug drawing is stripped from Shipping builds, but the CVar check itself is not; the `#if` removes both.
