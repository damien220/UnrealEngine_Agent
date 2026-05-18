# Unreal Build System — Reference

## Overview

Unreal uses **UnrealBuildTool (UBT)**, a custom C# build system that compiles modules, manages dependencies, and runs the Unreal Header Tool (UHT) for reflection code generation. It is not CMake, MSBuild, or Make — it has distinct conventions and failure modes.

---

## When to Run GenerateProjectFiles

Run `GenerateProjectFiles.bat` (Windows) or `GenerateProjectFiles.sh` (Linux/Mac) after any of these changes:

| Change | Why Regenerate? |
|---|---|
| Added/removed a source file (`.cpp`, `.h`) | IDE project files won't include the new file |
| Modified `.Build.cs` (module dependencies, defines) | Dependency graph changes aren't picked up by IDE |
| Modified `.uproject` (added/removed plugin) | Plugin module list is baked into project files |
| Added a new module directory | UBT won't scan it until project files are regenerated |
| Changed `DefaultEngine.ini` for module loading | Engine module load order changes |

**Failure to regenerate** causes IntelliSense errors, missing headers in the IDE, and confusing compile errors that seem unrelated to your change.

```
# Windows
%UE_ROOT%\Engine\Build\BatchFiles\GenerateProjectFiles.bat -project="YourGame.uproject" -game

# Linux
%UE_ROOT%/Engine/Build/BatchFiles/Linux/GenerateProjectFiles.sh -project="YourGame.uproject" -game
```

---

## Module Structure

Every module requires three files:

```
Source/
└── MyModule/
    ├── MyModule.Build.cs       ← dependency declaration (required)
    ├── MyModule.h              ← module header, declares MYGAME_API
    └── MyModule.cpp            ← IModuleInterface implementation
```

```csharp
// MyModule.Build.cs
public class MyModule : ModuleRules
{
    public MyModule(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

        PublicDependencyModuleNames.AddRange(new string[]
        {
            "Core", "CoreUObject", "Engine"
            // Add modules whose PUBLIC headers you include in your own PUBLIC headers
        });

        PrivateDependencyModuleNames.AddRange(new string[]
        {
            "Slate", "SlateCore"
            // Add modules whose headers you include only in .cpp files
        });

        // Optional: expose include paths to dependent modules
        // PublicIncludePaths.Add(Path.Combine(ModuleDirectory, "Public"));
    }
}
```

### Public vs Private Dependencies

| Type | When to use |
|---|---|
| `PublicDependencyModuleNames` | Module headers you include in your own `.h` files that other modules can see |
| `PrivateDependencyModuleNames` | Module headers used only in `.cpp` files — not exposed to dependent modules |

Using `PublicDependencyModuleNames` when `Private` would suffice increases transitive compilation scope. Default to `Private`, promote to `Public` only when your public headers include that module's types.

---

## Circular Module Dependencies

UBT enforces a strict directed acyclic graph for module dependencies. Circular dependencies **will cause link failures** — not compile errors, link failures, which are harder to diagnose.

```
// WRONG — Module A depends on Module B AND Module B depends on Module A
// Module A: PrivateDependencyModuleNames.Add("ModuleB")
// Module B: PrivateDependencyModuleNames.Add("ModuleA")
// Link error: unresolved external symbol or LINK : fatal error

// CORRECT patterns to break the cycle:
// 1. Extract shared types into a third module (MyModuleShared) that both A and B depend on
// 2. Use an interface module pattern — B depends on an interface module, A implements it
// 3. Use UE's delegate system to communicate between modules without direct dependency
// 4. Move the dependency direction — restructure so only A depends on B
```

**Diagnosis**: If you see `fatal error LNK1104` or `unresolved external` that appears after adding a module dependency, check for cycles with the `UBT -Graph` flag:

```
UnrealBuildTool.exe YourGame Win64 Development -Graph
```

---

## Reflection Macros

UHT processes reflection macros to generate `.generated.h` files before compilation. Missing, misplaced, or malformed macros cause **silent runtime failures**, not compile errors.

### Required macros and placement rules

```cpp
// UCLASS — must appear immediately before class definition
// The generated header must be the LAST include before the class
#include "MyActor.generated.h"  // Always last include

UCLASS(BlueprintType, Blueprintable)
class MYGAME_API AMyActor : public AActor
{
    GENERATED_BODY()  // Must be first line inside the class — no code before it
public:
    // ...
};
```

```cpp
// USTRUCT — same rules
USTRUCT(BlueprintType)
struct MYGAME_API FMyData
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere)
    float Value = 0.f;
};
```

```cpp
// UENUM — must use uint8 underlying type for Blueprint exposure
UENUM(BlueprintType)
enum class EMyState : uint8
{
    Idle    UMETA(DisplayName = "Idle"),
    Running UMETA(DisplayName = "Running"),
    Dead    UMETA(DisplayName = "Dead")
};
```

### Common silent failures from macro errors

| Mistake | Symptom |
|---|---|
| Missing `GENERATED_BODY()` | Class compiles but Blueprint subclassing crashes at runtime |
| `UPROPERTY` on non-`UObject` pointer type | Property editor appears correct; pointer is never tracked by GC |
| `UFUNCTION` on a function with unsupported parameter type | Function silently excluded from Blueprint; no error shown |
| `generated.h` not last include | UHT code gen conflicts; crashes on asset load |
| `UENUM` with non-`uint8` underlying type | Blueprint type picker shows garbage; asset corruption possible |

---

## Build Configuration Matrix

| Config | Optimizations | Logging | Checks | Use case |
|---|---|---|---|---|
| `Debug` | None | Full | `check` + `ensure` active | Deep debugging; slow |
| `DebugGame` | Engine optimized, game unoptimized | Full | Active | Day-to-day dev iteration |
| `Development` | Both optimized | Full | Active | Feature integration and QA |
| `Shipping` | Full | Stripped | `check` crashes; `ensure` is no-op | Release builds |
| `Test` | Shipping + test suite | Partial | Shipping rules | Automated test runs |

**Key consequence**: Code guarded only by `UE_BUILD_DEVELOPMENT` will not run in Shipping. Use `UE_BUILD_SHIPPING` guards for stripping, not `!UE_BUILD_DEVELOPMENT`.

---

## Plugin Modules

Plugins use the same `.Build.cs` structure but live under `Plugins/`:

```
Plugins/
└── MyPlugin/
    ├── MyPlugin.uplugin          ← plugin descriptor
    └── Source/
        └── MyPlugin/
            ├── MyPlugin.Build.cs
            ├── Public/
            └── Private/
```

In `.uplugin`:
```json
{
    "Modules": [
        {
            "Name": "MyPlugin",
            "Type": "Runtime",
            "LoadingPhase": "Default"
        }
    ]
}
```

**Loading phases**: `PreDefault` → `Default` → `PostDefault` → `PostEngineInit`. Use `PostEngineInit` only for modules that need engine subsystems fully initialized. Most game modules use `Default`.

---

## Common Build Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `error C2039: 'X' is not a member of 'Y'` after adding a module | Public/Private dependency mismatch | Move dependency to `PublicDependencyModuleNames` |
| `LINK : fatal error LNK1104` | Circular module dependency | Extract shared types to a third module |
| `'UMyClass' is not defined` in another module | Missing module dependency | Add the defining module to `.Build.cs` |
| `error: no member named 'GENERATED_BODY'` | Missing or wrong `.generated.h` include | Add `#include "MyClass.generated.h"` as the last include |
| Missing Blueprint functions after adding `UFUNCTION` | UHT not re-run | Rebuild — UHT runs automatically on build, not just compile |
| Plugin headers not found | Plugin not enabled in `.uproject` | Add plugin to `Plugins` array in `.uproject` with `"Enabled": true` |
