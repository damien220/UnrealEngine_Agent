# Engine Version Macros — Reference

## Overview

Multi-version UE projects (shipping on UE4.26 and UE5.x simultaneously, or migrating gradually) require preprocessor guards that correctly compare engine versions across the UE4→UE5 boundary. The naive approach of comparing `ENGINE_MINOR_VERSION` breaks at the transition because UE5 reset the minor version back to 0. Use the dedicated comparison macros from `EngineVersionComparison.h` exclusively.

---

## Core Version Number Macros

**Defined internally via:** `Runtime/Launch/Resources/Version.h`

> **Do not** `#include "Runtime/Launch/Resources/Version.h"` directly and do **not** add `"Launch"` to `.Build.cs` — both cause linker errors. The macros are already injected into every translation unit by UBT.

| Macro | Type | Example value |
|---|---|---|
| `ENGINE_MAJOR_VERSION` | Integer | `5` |
| `ENGINE_MINOR_VERSION` | Integer | `3` |
| `ENGINE_PATCH_VERSION` | Integer | `2` |
| `ENGINE_IS_PROMOTED_BUILD` | Bool (0/1) | `1` for released builds |
| `BUILT_FROM_CHANGELIST` | Integer | `0` for source builds, non-zero for Epic releases |

### Why raw comparison is unsafe

```cpp
// WRONG — Breaks at the UE4→UE5 boundary.
// UE4.27: MAJOR=4, MINOR=27 → passes
// UE5.0:  MAJOR=5, MINOR=0  → fails silently because MINOR reset to 0
#if ENGINE_MINOR_VERSION >= 20
    // Intended: "UE 4.20 or newer" — actually fires on UE5.0 only if MINOR >= 20
#endif

// WRONG — Still fragile for cross-major comparisons
#if ENGINE_MAJOR_VERSION == 5 && ENGINE_MINOR_VERSION >= 1
    // Fine for UE5-only projects, but fails to account for UE4.x gracefully
#endif
```

---

## Recommended: `EngineVersionComparison.h` Macros

**Header:** `Misc/EngineVersionComparison.h`  
**Availability:** UE 4.19 and all UE5.x versions

```cpp
#include "Misc/EngineVersionComparison.h"
```

### Macros provided

| Macro | Meaning | "At least" equivalent |
|---|---|---|
| `UE_VERSION_OLDER_THAN(Major, Minor, Patch)` | True if engine < specified version | `!UE_VERSION_OLDER_THAN(...)` |
| `UE_VERSION_NEWER_THAN(Major, Minor, Patch)` | True if engine > specified version | `UE_VERSION_NEWER_THAN(...)` |

> `UE_VERSION_AT_LEAST` does **not** exist in the engine — invert `UE_VERSION_OLDER_THAN` for "at least" semantics.

### Correct multi-version branching pattern

```cpp
#include "Misc/EngineVersionComparison.h"

// Branch across UE4 / UE5 boundary safely
#if UE_VERSION_OLDER_THAN(5, 0, 0)
    // UE4.x code path (any 4.x version)
    FVector Location = Actor->GetActorLocation();
    float X = Location.X;  // Safe: float in UE4
#else
    // UE5.0+ code path
    FVector Location = Actor->GetActorLocation();
    double X = Location.X;  // Safe: double in UE5 (LWC)
#endif

// "At least UE5.1" guard
#if !UE_VERSION_OLDER_THAN(5, 1, 0)
    // UE5.1+ specific API
    UE_LOGFMT(LogMyGame, Log, "Structured log available since 5.1");
#endif

// Version range: UE4.26 through UE4.27 only
#if !UE_VERSION_OLDER_THAN(4, 26, 0) && UE_VERSION_OLDER_THAN(5, 0, 0)
    // Applies only to UE 4.26.x and 4.27.x
#endif
```

---

## Build System Version Checking (.Build.cs)

When a C++ symbol needs to differ per engine version, define a custom preprocessor flag in `.Build.cs` rather than repeating the version comparison at every call site.

```csharp
// In MyModule.Build.cs — ReadOnlyTargetRules Target is available
public class MyModule : ModuleRules
{
    public MyModule(ReadOnlyTargetRules Target) : base(Target)
    {
        // Access engine version
        int MajorVersion = Target.Version.MajorVersion;
        int MinorVersion = Target.Version.MinorVersion;
        int PatchVersion = Target.Version.PatchVersion;

        // Define symbols consumed in C++ via #if
        if (MajorVersion >= 5)
        {
            PublicDefinitions.Add("MY_ENGINE_UE5_PLUS=1");
        }
        else
        {
            PublicDefinitions.Add("MY_ENGINE_UE5_PLUS=0");
        }

        // Fine-grained minor version gates
        if (MajorVersion == 5 && MinorVersion >= 2)
        {
            PublicDefinitions.Add("MY_ENGINE_HAS_STRUCTURED_LOG=1");
        }
    }
}
```

```cpp
// C++ usage of the custom symbols
#if MY_ENGINE_UE5_PLUS
    // UE5-specific path
#else
    // UE4 path
#endif
```

**Note:** `PublicDefinitions` replaces the deprecated `Definitions` API (removed before UE4.19). Never use `Definitions.Add(...)`.

---

## UE4 → UE5 Breaking Changes Requiring Version Guards

### FVector precision (Large World Coordinates)

```cpp
#include "Misc/EngineVersionComparison.h"

void StorePosition(const FVector& V)
{
#if UE_VERSION_OLDER_THAN(5, 0, 0)
    float X = V.X;  // FVector.X is float in UE4
    float Y = V.Y;
    float Z = V.Z;
#else
    double X = V.X;  // FVector.X is double in UE5 (LWC enabled by default)
    double Y = V.Y;
    double Z = V.Z;
#endif
}

// Prefer the FVector typedef aliases — they adapt automatically:
FVector::FReal X = V.X;  // Resolves to float (UE4) or double (UE5) — safest approach
```

### Structured logging (UE5.2+)

```cpp
// All versions — always available
UE_LOG(LogMyGame, Warning, TEXT("Value: %d"), MyInt);

// UE5.2+ only — compile-error on older versions if not guarded
#if !UE_VERSION_OLDER_THAN(5, 2, 0)
#include "Logging/StructuredLog.h"
UE_LOGFMT(LogMyGame, Log, "Value {0}", MyInt);
#endif
```

### `TObjectPtr` introduction (UE5.0+)

```cpp
// UE4: raw pointer in UPROPERTY is the only option
// UE5: TObjectPtr<T> is the preferred UPROPERTY member type (tracked pointer wrapper)
#if UE_VERSION_OLDER_THAN(5, 0, 0)
    UPROPERTY()
    UMyComponent* MyComp;
#else
    UPROPERTY()
    TObjectPtr<UMyComponent> MyComp;  // Preferred in UE5
#endif
```

### `GetTypeHash` / hashing API changes

Some hash and type-trait APIs were modified between UE4.27 and UE5.0. When porting, wrap affected sections and verify against the engine changelog for your specific minor versions.

---

## Deprecation Guards

Use deprecation pragmas to suppress warnings when intentionally calling deprecated APIs in bridging code.

```cpp
PRAGMA_DISABLE_DEPRECATION_WARNINGS
    // Call deprecated API — compiler suppresses C4996 / -Wdeprecated here
    OldApi_DoSomething();
PRAGMA_ENABLE_DEPRECATION_WARNINGS
```

**Marking your own APIs as deprecated:**

```cpp
// Deprecate a function — available all versions
UE_DEPRECATED(5.1, "Use NewFunction() instead — OldFunction will be removed in 5.3")
virtual void OldFunction();

// Deprecate an entire class
UCLASS()
class UE_DEPRECATED(5.1, "Use UNewSystem instead") MYGAME_API UOldSystem : public UObject
{
    GENERATED_BODY()
};
```

`UE_DEPRECATED` fires a compiler warning in Debug/Development/Shipping for all callers. Wrap call sites with `PRAGMA_DISABLE_DEPRECATION_WARNINGS` if you must keep calling a deprecated internal API during a migration window.

---

## What You Cannot Wrap in Version Guards

UHT (Unreal Header Tool) processes reflection macros before the C++ compiler sees them — preprocessor conditionals are **not evaluated** by UHT. The following macros must appear unconditionally:

```cpp
// WRONG — UHT will silently skip the UPROPERTY, causing GC to miss it
#if !UE_VERSION_OLDER_THAN(5, 0, 0)
    UPROPERTY()
    TObjectPtr<UMyComponent> MyComp;
#endif

// CORRECT — declare unconditionally; version-gate the implementation
UPROPERTY()
UMyComponent* MyComp;  // Use the UE4-compatible type; assign correctly at runtime
```

**The one exception:** `WITH_EDITORONLY_DATA` is a UHT-aware macro specifically designed for conditionally stripping editor-only `UPROPERTY` fields from cook builds — it is the only preprocessor guard UHT handles correctly around property declarations.

---

## Platform Macros — Common Co-occurrences

These platform macros are often combined with version guards for platform+version-specific code paths. Defined in `HAL/Platform.h` (no include needed — always injected by UBT).

| Macro | When true |
|---|---|
| `PLATFORM_WINDOWS` | Windows target |
| `PLATFORM_MAC` | macOS target |
| `PLATFORM_LINUX` | Linux target |
| `PLATFORM_CONSOLE` | Any console (PS5, Xbox Series) |
| `WITH_EDITOR` | Editor-included builds (not Shipping) |
| `UE_BUILD_SHIPPING` | Shipping config |
| `UE_BUILD_DEVELOPMENT` | Development config |
| `UE_BUILD_DEBUG` | Debug config (rarely set — test with `!UE_BUILD_SHIPPING` instead) |

```cpp
// Platform + version guard
#if PLATFORM_WINDOWS && !UE_VERSION_OLDER_THAN(5, 2, 0)
    // Windows-specific, UE5.2+ API
#endif

// Editor-only code with version gate
#if WITH_EDITOR && !UE_VERSION_OLDER_THAN(5, 1, 0)
    // Editor tool that uses a UE5.1+ editor API
#endif
```

---

## Quick Decision Table

| Goal | Pattern |
|---|---|
| Code for UE5.0+ only | `#if !UE_VERSION_OLDER_THAN(5, 0, 0)` |
| Code for UE4.x only | `#if UE_VERSION_OLDER_THAN(5, 0, 0)` |
| Code for UE4.26 through UE4.27 | `#if !UE_VERSION_OLDER_THAN(4, 26, 0) && UE_VERSION_OLDER_THAN(5, 0, 0)` |
| Code for exactly UE5.3 | `#if !UE_VERSION_OLDER_THAN(5, 3, 0) && UE_VERSION_OLDER_THAN(5, 4, 0)` |
| Build-time symbol for C++ | `PublicDefinitions.Add("MY_FLAG=1")` in `.Build.cs` |
| Suppress deprecation warning | Bracket with `PRAGMA_DISABLE_DEPRECATION_WARNINGS` |
| Mark own API deprecated | `UE_DEPRECATED(5.x, "Message")` before declaration |
