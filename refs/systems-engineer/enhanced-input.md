# Enhanced Input System — Reference

## Overview

Enhanced Input (EI) replaced UE's legacy input system as of UE5.0 and is the mandatory standard in UE5.1+. It uses data-driven `UInputAction` and `UInputMappingContext` assets instead of the legacy `DefaultInput.ini` axis/action bindings. Legacy input still compiles in UE5 but is unsupported for new projects.

---

## Migration from Legacy Input

### Disable legacy input (new projects)
In `DefaultEngine.ini` (or Project Settings > Input):
```ini
[/Script/Engine.InputSettings]
DefaultPlayerInputClass=/Script/EnhancedInput.EnhancedPlayerInput
DefaultInputComponentClass=/Script/EnhancedInput.EnhancedInputComponent
```
Without these two settings, `Cast<UEnhancedInputComponent>` in `SetupPlayerInputComponent` returns null and all Enhanced Input bindings silently fail.

### Build.cs dependency
```csharp
PublicDependencyModuleNames.AddRange(new string[] {
    "EnhancedInput"
});
```

### Required headers
```cpp
#include "EnhancedInputComponent.h"           // UEnhancedInputComponent
#include "EnhancedInputSubsystems.h"           // UEnhancedInputLocalPlayerSubsystem
#include "InputMappingContext.h"               // UInputMappingContext
#include "InputAction.h"                       // UInputAction
```

---

## Core Assets

### UInputAction
Represents one logical action (Jump, Fire, Move, Look). Configure in the editor:

| Property | Options | Use |
|---|---|---|
| **Value Type** | `Digital (bool)` | Button press/release |
| | `Axis1D (float)` | Single-axis (mouse wheel, gamepad trigger) |
| | `Axis2D (FVector2D)` | Two-axis (mouse look, gamepad stick) |
| | `Axis3D (FVector)` | Three-axis (6DOF devices) |
| **Triggers** | `Pressed`, `Released`, `Hold`, etc. | Default behavior for this action |
| **Modifiers** | `Dead Zone`, `Scale`, etc. | Applied before triggers evaluate |

### UInputMappingContext
Maps physical keys/buttons to `UInputAction` assets. One context per control scheme (Gameplay, UI, Vehicle). Contexts stack by priority — higher priority blocks lower priority bindings for the same key.

---

## Adding/Removing Contexts at Runtime

```cpp
// AMyCharacter.h
UPROPERTY(EditDefaultsOnly, Category = "Input")
TObjectPtr<UInputMappingContext> GameplayContext;

UPROPERTY(EditDefaultsOnly, Category = "Input")
TObjectPtr<UInputMappingContext> UIContext;

// AMyCharacter.cpp — add gameplay context on BeginPlay
void AMyCharacter::BeginPlay()
{
    Super::BeginPlay();

    if (APlayerController* PC = Cast<APlayerController>(GetController()))
    {
        if (UEnhancedInputLocalPlayerSubsystem* Subsystem =
            ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer()))
        {
            Subsystem->AddMappingContext(GameplayContext, 0); // Priority 0 = lowest
        }
    }
}

// Swap to UI context when a menu opens
void AMyCharacter::OpenMenu()
{
    if (UEnhancedInputLocalPlayerSubsystem* Subsystem = GetInputSubsystem())
    {
        Subsystem->RemoveMappingContext(GameplayContext);
        Subsystem->AddMappingContext(UIContext, 1); // Priority 1 blocks gameplay
    }
}
```

**Priority rule**: When two active contexts bind the same key, the context with the higher priority wins. The lower-priority binding is **not consumed** — it is blocked. Remove contexts that should no longer fire rather than relying on priority alone.

---

## Binding Actions in C++

```cpp
void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    // MUST cast — UInputComponent* in the function signature is the legacy type
    UEnhancedInputComponent* EIC = Cast<UEnhancedInputComponent>(PlayerInputComponent);
    if (!EIC)
    {
        UE_LOG(LogMyGame, Error, TEXT("EnhancedInputComponent not found — "
            "check DefaultPlayerInputClass in DefaultEngine.ini"));
        return;
    }

    // Digital action (bool) — fires on ETriggerEvent::Triggered (each frame held down)
    EIC->BindAction(JumpAction, ETriggerEvent::Started, this, &AMyCharacter::OnJumpStarted);
    EIC->BindAction(JumpAction, ETriggerEvent::Completed, this, &AMyCharacter::OnJumpReleased);

    // Axis2D action — FVector2D in callback
    EIC->BindAction(MoveAction, ETriggerEvent::Triggered, this, &AMyCharacter::OnMove);

    // Axis2D action — look / camera rotation
    EIC->BindAction(LookAction, ETriggerEvent::Triggered, this, &AMyCharacter::OnLook);
}

// Callback signatures must match the action's Value Type exactly
void AMyCharacter::OnJumpStarted(const FInputActionValue& Value)
{
    // Value.Get<bool>() for Digital actions
    Jump();
}

void AMyCharacter::OnMove(const FInputActionValue& Value)
{
    // Value.Get<FVector2D>() for Axis2D actions
    FVector2D MoveInput = Value.Get<FVector2D>();
    AddMovementInput(GetActorForwardVector(), MoveInput.Y);
    AddMovementInput(GetActorRightVector(),   MoveInput.X);
}

void AMyCharacter::OnLook(const FInputActionValue& Value)
{
    FVector2D LookInput = Value.Get<FVector2D>();
    AddControllerYawInput(LookInput.X);
    AddControllerPitchInput(LookInput.Y);
}
```

---

## Input Modifiers

Applied to the raw input value before trigger evaluation. Stack multiple modifiers on one action.

| Modifier | Effect | Common use |
|---|---|---|
| `Dead Zone` | Ignores input below threshold | Gamepad stick drift prevention |
| `Scale` | Multiplies by scalar | Mouse sensitivity, axis inversion |
| `Negate` | Flips sign | Inverted look option |
| `Swizzle Input Axis Values` | Reorders XYZ | **Required for Axis2D from two 1D keys** (see below) |
| `FOV Scaling` | Scales by camera FOV | Aim-down-sights mouse feel |
| `Smooth` | Lerps input over time | Reduce jittery gamepad |

### Axis2D from two separate keys (WASD)

The most common gotcha: binding `W/S` to a Move action with Axis2D returns only `Y`. `A/D` binds return only `X`. Without `Swizzle`, both sets write to the X channel and cancel each other.

```
In UInputMappingContext:
  MoveAction ← W key   : Modifiers = [Scale (1.0)]              → contributes Y as +1
  MoveAction ← S key   : Modifiers = [Negate, Scale (1.0)]      → contributes Y as -1
  MoveAction ← D key   : Modifiers = [Swizzle (YXZ), Scale(1)]  → moves value to X channel
  MoveAction ← A key   : Modifiers = [Swizzle (YXZ), Negate]    → X channel negative
```

Without `Swizzle Input Axis Values` on the D/A keys, `MoveInput.X` is always 0.

---

## Input Triggers

| Trigger | When it fires | Use |
|---|---|---|
| `Pressed` | Single frame on key down | Jump, shoot (one shot) |
| `Released` | Single frame on key up | Release charge, open menu |
| `Hold` | Each frame after `HoldTimeThreshold` (default 0.5s) | Charged attack |
| `HoldAndRelease` | On key up if held ≥ threshold | Confirm a held interaction |
| `Tap` | On key up if released < `TapReleaseTimeThreshold` | Quick-tap vs hold disambiguation |
| `Pulse` | Fires at intervals while held | Rapid-fire while button held |
| `Chorded Action` | Fires only when another action is also active | Combo: Shift+E |

Multiple triggers can be added to one action binding — they all must pass for the event to fire (AND logic). To implement OR logic, create two separate bindings.

---

## Exposing Input Assets as UPROPERTY (Designer Override)

```cpp
// AMyCharacter.h — expose all input assets to Blueprint/editor
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
TObjectPtr<UInputMappingContext> DefaultMappingContext;

UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
TObjectPtr<UInputAction> JumpAction;

UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
TObjectPtr<UInputAction> MoveAction;

UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
TObjectPtr<UInputAction> LookAction;

UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Input")
TObjectPtr<UInputAction> SprintAction;
```

Assign these in the Blueprint subclass's default properties. This allows designers to swap input assets per character archetype without touching C++.

---

## ETriggerEvent Reference

| ETriggerEvent | Meaning |
|---|---|
| `None` | Trigger not reached |
| `Triggered` | Fires every frame the trigger condition is met (use for Axis/Hold) |
| `Started` | Fires on the first frame the trigger was met (use for Pressed) |
| `Ongoing` | Fires each frame the trigger is still evaluating (between Started and Triggered) |
| `Completed` | Fires when the trigger condition was met but is now released |
| `Canceled` | Trigger was in progress but was interrupted (e.g., Hold canceled by key release before threshold) |

For digital actions like Jump, bind to `Started` (not `Triggered`) — `Triggered` fires every frame while held, flooding the jump call.

---

## Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `DefaultPlayerInputClass` not set to Enhanced | `Cast<UEnhancedInputComponent>` returns null; no bindings fire | Set in `DefaultEngine.ini` |
| Context not added in `BeginPlay` | Actions exist but never fire | `AddMappingContext` in `BeginPlay` after `Super::` |
| Axis2D without Swizzle on A/D keys | `MoveInput.X` always 0 | Add `Swizzle (YXZ)` modifier to horizontal key bindings |
| Callback signature uses `float` instead of `FInputActionValue&` | Compile error or silent bind failure | Match callback to action Value Type |
| Removing context before adding new one crashes | Can happen on rapid context swap | Check `HasMappingContext` before Remove, or use priority instead |
| `BindAction` called before `SetDefaultInputComponentClass` set | Binds silently succeed but events never fire | Verify `DefaultEngine.ini` first |
