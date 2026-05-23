# Animation Blueprint: Update Order, State Machines, and Networking

## UAnimInstance Update Order

Animation blueprints in UE5 follow a strict multi-threaded update order. Understanding the threading model is essential for correct logic placement and performance.

### Per-Frame Execution Order

1. **`NativeInitializeAnimation()`** — Called once when the animation instance is created. Safe place to initialize timers, cache component references, and set default values.

2. **`NativeUpdateAnimation(float DeltaSeconds)`** — Called on the **game thread** every frame *before* animation nodes evaluate. This is where you:
   - Read `AActor` and `UCharacter` properties (safe, main thread)
   - Fetch velocity, acceleration, directional input
   - Compute complex logic (foot IK calculations, animation state machines)
   - Write to `UPROPERTY` member variables that animation nodes read
   - Perform all expensive calculations

   ```cpp
   void UMyCharacterAnimBP::NativeUpdateAnimation(float DeltaSeconds)
   {
       Super::NativeUpdateAnimation(DeltaSeconds);
       
       if (AMyCharacter* Char = Cast<AMyCharacter>(TryGetPawnOwner()))
       {
           CharacterSpeed = Char->GetCharacterMovement()->Velocity.Length();
           bIsGrounded = Char->GetCharacterMovement()->IsMovingOnGround();
           LookDirection = Char->GetControlRotation().Vector();
       }
   }
   ```

3. **`NativeThreadSafeUpdateAnimation(float DeltaSeconds)`** — Called on an **animation worker thread** after `NativeUpdateAnimation()` completes. Use this for:
   - Expensive blending math (inverse kinematics, procedural animation)
   - Complex state machine evaluations
   - Work that doesn't depend on UObject changes

   **Critical rule**: Do NOT access `AActor` properties, `UCharacter`, or any UObjects here. Only read the `UPROPERTY` values you set in `NativeUpdateAnimation()`.

   ```cpp
   void UMyCharacterAnimBP::NativeThreadSafeUpdateAnimation(float DeltaSeconds)
   {
       Super::NativeThreadSafeUpdateAnimation(DeltaSeconds);
       
       // Safe: reading UPROPERTY values set above
       float BlendAlpha = FMath::Clamp(CharacterSpeed / MaxSpeed, 0.0f, 1.0f);
       ComputedAimOffset = FMath::Lerp(FVector::ZeroVector, TargetAimOffset, BlendAlpha);
       
       // NOT SAFE (would crash): accessing actor properties
       // float Health = TryGetPawnOwner()->GetHealth();  // CRASH
   }
   ```

### Animation Node Evaluation

After both Update functions complete, animation nodes (state machines, blend spaces, sequence players) evaluate their inputs (all `UPROPERTY` member variables). Nodes that read UObject properties directly will fail or cause stuttering.

---

## Animation State Machine Design

### Entry State Rule

The **Entry State** (specified in state machine details) is always the first state entered when the state machine initializes. There is no transition rule for entry — the state is automatically set.

```cpp
// In animation blueprint designer:
// Right-click state machine → Details → Search "Entry" → select your idle state
```

### Transition Rules Must Be Pure

Transition rules are evaluated **every frame** and must be **pure functions** (no side effects). A pure function:
- Has no output execution pins (no branching)
- Returns only a boolean
- Reads variables without modifying state

**Correct transition rule** (pure):
```cpp
// In animation blueprint, connection from State A → State B
// Transition Rule: pure function that returns bool
// Speed > 100.0
// (This is a simple comparison with no side effects)
```

**Incorrect transition rule** (not pure):
```cpp
// Calling a function that modifies variables
// Incrementing counters
// Spawning effects
// These break the assumption that rules are re-evaluated every frame
```

### Timing Transitions

Use these functions to control transition timing:

```cpp
// In state machine or anim notify:
float TimeRemaining = GetRelevantAnimTimeRemaining();
float CurrentTime = GetRelevantAnimTimeElapsed();

// Transition only in the last 0.5 seconds of animation
if (TimeRemaining < 0.5f && Condition)
{
    // Allow transition
}
```

### Handling Missing Animations

Guard against missing animation sequences:

```cpp
// In transition rule (pure function):
bool CanTransitionToAttack()
{
    if (bTryGetRelevantAnimSequenceFailed)
    {
        return false;  // Animation not set, block transition
    }
    return AttackInput && !bIsAttacking;
}
```

---

## Animation Layers and Modular Blueprints

### UAnimLayerInterface Declaration

`UAnimLayerInterface` defines the "slots" that other animation blueprints can implement. Create one as a Blueprint or C++ class:

```cpp
// C++ Header
UCLASS(Blueprintable)
class YOURMODULE_API UCharacterAnimLayerInterface : public UAnimLayerInterface
{
    GENERATED_BODY()

public:
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable)
    void FullBody();
    
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable)
    void UpperBody();
    
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable)
    void UpperBodyAimed();
};
```

### Linked Layer Nodes

In the main animation blueprint, add a **Linked Anim Layer** node for each layer interface:

```
Animation Blueprint Designer:
[Linked Anim Layer]
  ├─ Instance Class: UCharacterAnimLayerInterface
  ├─ Input Pin: [connected to state machine output]
  └─ (Outputs: FullBody, UpperBody, UpperBodyAimed)
```

In the Linked Layer node details, set:
- **Instance Class**: Your layer interface class
- **Linked Anim Class** (optional): Default implementation (can be overridden at runtime)

### Runtime Layer Swapping

At runtime, swap the implementation of a layer:

```cpp
void AMyCharacter::EquipWeapon(AWeapon* NewWeapon)
{
    if (UAnimInstance* AnimInst = GetMesh()->GetAnimInstance())
    {
        // Link the upper-body layer to weapon-specific anim BP
        TSubclassOf<UAnimInstance> WeaponAnimLayer = NewWeapon->GetUpperBodyAnimBPClass();
        AnimInst->LinkAnimClassLayers(WeaponAnimLayer);
    }
}

void AMyCharacter::UnequipWeapon()
{
    if (UAnimInstance* AnimInst = GetMesh()->GetAnimInstance())
    {
        // Reset to default implementation
        AnimInst->UnlinkAnimClassLayers();
    }
}
```

**Key points**:
- `LinkAnimClassLayers()` immediately executes the linked blueprint
- `UnlinkAnimClassLayers()` resets to the original/default
- Linking is cheap (no recompilation) and can happen mid-game

---

## Animation Fast Path Optimization

### What Breaks Fast Path

The **animation fast path** allows animation nodes to read `UPROPERTY` member variables directly without Blueprint Virtual Machine execution. This optimization breaks if:

1. **Non-const function calls** in the animation graph (Blueprint functions)
2. **Object pins** (references to actors, components)
3. **Complex computed expressions** (math not directly on UPROPERTY values)

### Enabling Fast Path

Structure your animation blueprint to only read `UPROPERTY` values:

```cpp
// Header
UPROPERTY(EditAnywhere, Category = "Animation")
float CharacterSpeed = 0.0f;

UPROPERTY(EditAnywhere, Category = "Animation")
bool bIsGrounded = false;

UPROPERTY(EditAnywhere, Category = "Animation")
EMovementDirection MovementDirection = EMovementDirection::Forward;

// NativeUpdateAnimation sets these
void UMyCharacterAnimBP::NativeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeUpdateAnimation(DeltaSeconds);
    if (AMyCharacter* Char = Cast<AMyCharacter>(TryGetPawnOwner()))
    {
        CharacterSpeed = Char->GetCharacterMovement()->Velocity.Length();
        bIsGrounded = Char->GetCharacterMovement()->IsMovingOnGround();
        MovementDirection = ComputeMovementDirection(Char);
    }
}
```

In the animation graph, connect these `UPROPERTY` variables directly to blend spaces and sequence players (no Blueprint function calls):

```
Animation Graph:
[CharacterSpeed] ──→ [BlendSpace 1D: Idle-Walk-Run]
[bIsGrounded] ──────→ [Gate]
[MovementDirection]─→ [BlendSpace 2D: Directional Strafing]
```

**Fast path enabled**: Nodes read `CharacterSpeed`, `bIsGrounded`, `MovementDirection` directly from memory.

**Fast path disabled**: If you added a Blueprint function like `GetAdjustedSpeed()` between the property and the blend space, the node must call the Blueprint VM, breaking optimization.

### FAnimNode_SequencePlayer Fast Path

Sequence players break fast path if:
- The playback rate is a computed value (use a `UPROPERTY float` instead)
- The sequence reference changes (set in `NativeUpdateAnimation()`, not dynamically)
- Any property pin is connected to a Blueprint function output

**Correct fast path**:
```cpp
UPROPERTY(EditAnywhere, Category = "Animation")
float PlayRate = 1.0f;

void UMyCharacterAnimBP::NativeUpdateAnimation(float DeltaSeconds)
{
    PlayRate = CharacterSpeed / DefaultWalkSpeed;  // Clamped value written to UPROPERTY
}

// In animation graph:
// [Sequence Player: Run] → Input: PlayRate = [PlayRate property, direct read]
```

---

## Root Motion

### Setup

1. **On the animation montage or sequence**:
   - **Anim Sequence**: Check `Enable Root Motion` in details
   - **Anim Montage**: Slot nodes must have `bUseSlotNodeWeight = true`

2. **On the Character Movement Component**:
   ```cpp
   CharacterMovement->bEnablePhysicsInteraction = false;
   CharacterMovement->MovementState = EMovementMode::MOVE_Walking;  // or custom mode
   ```

3. **In code**, set the root motion mode on the character:
   ```cpp
   // Apply root motion from animation
   CharacterMovement->CharacterOwner->CharacterMovement->RootMotionMode = ERootMotionMode::RootMotionFromEverything;
   ```

### Networked Root Motion (Multiplayer)

**Server-authoritative model**:
- Server plays the animation and extracts root motion locally
- Server replicates the resulting **position**, not the root motion itself
- Clients receive the final position and play the same animation (assuming synchronized animation state)

```cpp
// Server
void AMyCharacter::PlayAttackMontage()
{
    if (GetLocalRole() == ROLE_Authority)
    {
        GetMesh()->GetAnimInstance()->Montage_Play(AttackMontage);
        MulticastPlayAttackMontage();
    }
}

UFUNCTION(NetMulticast, Unreliable)
void AMyCharacter::MulticastPlayAttackMontage()
{
    if (GetLocalRole() != ROLE_Authority)
    {
        GetMesh()->GetAnimInstance()->Montage_Play(AttackMontage);
    }
}
```

**Key insight**: Root motion is computed from the animation on each client independently. The server ensures animation sync and monitors the final position; clients trust the animation playback.

---

## Common Anim Notify and Montage Patterns

### Animation Montage System

```cpp
// Play a montage
UAnimMontage* Montage = GetMontage();
GetMesh()->GetAnimInstance()->Montage_Play(Montage, PlayRate, EMontagePlayReturnType::Duration);

// Stop all montages
GetMesh()->GetAnimInstance()->StopAllMontages(BlendOutTime);

// Bind to completion
FOnMontageEnded MontageEndedDelegate;
MontageEndedDelegate.BindDynamic(this, &AMyCharacter::OnMontageEnded);
GetMesh()->GetAnimInstance()->Montage_SetEndDelegate(MontageEndedDelegate, Montage);
```

### Slot Nodes and Interrupt Points

Montages use **slot nodes** in the animation graph to determine where the montage is blended in:

```
Animation Graph:
[Idle State] ──→ [Slot: "DefaultSlot"] ──→ [Output Pose]

// The slot determines blend-in and blend-out timing
// Multiple montages can't play in the same slot simultaneously
```

---

## Performance Guidelines

| Rule | Impact | Solution |
|------|--------|----------|
| Complex logic in transition rules | Per-frame evaluation | Move to `NativeUpdateAnimation()`, write `UPROPERTY`, read in rule |
| UObject access in `NativeThreadSafeUpdateAnimation()` | Crash / race condition | Move to `NativeUpdateAnimation()` |
| Animation function calls in anim graph | Disables fast path | Read `UPROPERTY` directly |
| Non-pure transition rules | Unpredictable state | Use pure functions only (no state modification) |
| Missing anim montage on play | Silent failure / wrong animation | Use `bTryGetRelevantAnimSequenceFailed` guard |
| Unlinked animation layers at runtime | Wrong character animation | Pre-link default layer in Blueprint |

---

## Summary

- **Update order**: `NativeUpdateAnimation()` (game thread) → `NativeThreadSafeUpdateAnimation()` (worker thread) → node evaluation
- **State machines**: Entry state automatic; transition rules pure; use `GetRelevantAnimTimeRemaining()` for timing
- **Layers**: `UAnimLayerInterface` defines slots; `LinkAnimClassLayers()` swaps implementations at runtime
- **Fast path**: Read `UPROPERTY` directly in anim graph; avoid Blueprint function calls
- **Root motion**: Server-authoritative; animation plays on all clients; position replicates from server
- **Montages**: Play via `Montage_Play()`; use slot nodes for blending; bind to `OnMontageEnded`

