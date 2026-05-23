# Gameplay Ability System: Blueprint Design and Task Nodes

## UGameplayAbility: C++ vs Blueprint

The decision to implement an ability in C++ or Blueprint depends on complexity, performance sensitivity, and reusability.

### Blueprint-Suitable Abilities

Use Blueprint when:
- Ability logic is straightforward (simple sequencing, basic conditions)
- Ability is one-off or game-specific (not reused across projects)
- Designers need to iterate rapidly (C++ recompile delays block iteration)

**Correct approach**: Inherit from `UGameplayAbility` in Blueprint, implement `ActivateAbility` and `EndAbility` events.

```
Blueprint Class: BP_FireWeaponAbility
├─ Parent Class: UGameplayAbility
├─ Event: ActivateAbility
│  ├─ [Wait for Input] ──→ [Fire]
│  └─ [Play Montage] ──→ [Apply Damage] ──→ [End Ability]
└─ Event: EndAbility
   └─ [Cleanup]
```

### C++-Required Abilities

Use C++ when:
- Complex activation conditions (nested checks, caching)
- Override `CanActivateAbility()` with non-trivial logic
- Performance-critical abilities (used frequently, must minimize Blueprint VM calls)
- Abilities need custom replication or prediction behavior

```cpp
class UMyComplexAbility : public UGameplayAbility
{
    UFUNCTION(BlueprintOverride)
    bool CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, 
                            const FGameplayTagContainer* RelevantTags, OUT FGameplayTagContainer* OptionalRelevantTags) const override
    {
        // Complex logic: resource checks, cooldown, target validation
        if (!Super::CanActivateAbility(Handle, ActorInfo, RelevantTags, OptionalRelevantTags))
            return false;
        
        if (!HasValidTarget())
            return false;
        
        if (!ConsumeResource(GetRequiredResourceCost()))
            return false;
        
        return true;
    }
};
```

---

## Ability Task Nodes in Blueprint

Ability Tasks are latent nodes (with output execution pins that fire asynchronously) that allow abilities to execute across multiple frames. Each task represents a distinct unit of work.

### Common Task Types

#### WaitGameplayEvent

Waits until a gameplay event with a specific tag is fired:

```
Blueprint:
[Activate Ability]
  ├─ [Wait Gameplay Event (EventTag="Ability.Attack.HitConfirmed")] ──(On Event)──→ [Apply Damage]
  │                                                                   ──(On Cancel)─→ [End Ability]
  └─ [Play Montage]
```

The task looks at the owning ability's ASC (Ability System Component) by default. If you want to wait for an event from another actor, pass that actor as the target.

```cpp
// C++ usage (for reference)
UAbilityTask_WaitGameplayEvent* Task = UAbilityTask_WaitGameplayEvent::WaitGameplayEvent(
    this,
    FGameplayTag::RequestGameplayTag(FName("Ability.Attack.HitConfirmed")),
    nullptr,  // TargetActor (nullptr = owning ASC)
    true      // bOnlyTriggerOnce
);
Task->EventReceived.AddDynamic(this, &UGameplayAbility::OnEventReceived);
```

#### WaitGameplayTagAdded / WaitGameplayTagRemoved

Waits until a specific tag is added to or removed from the owning character:

```
Blueprint:
[Wait Gameplay Tag Added (Tag="Character.Status.Stunned")] ──(On Tag Added)──→ [Stop Ability]
[Wait Gameplay Tag Removed (Tag="Character.Status.Stunned")] ──(On Tag Removed)──→ [Resume Ability]
```

Used for status effect interrupts:

```
Blueprint:
[Activate Ability]
  ├─ [Wait Gameplay Tag Added ("Character.Status.StunImmune")] ──(On Added)──→ [End Ability Early]
  └─ [Execute Ability Logic]
```

#### WaitDelay

Waits for a specified duration (in seconds):

```
Blueprint:
[Play Impact VFX]
  ├─ [Wait Delay (0.5 seconds)] ──→ [Play Secondary VFX]
  └─ [Play SFX]
```

Simple timer task; output pin fires after delay expires.

#### WaitTargetData

Waits for player input to select a target or location:

```
Blueprint:
[Activate Ability]
  ├─ [Wait Target Data (TargetingClass=BP_AoETargeting)] ──(On Target Confirmed)──→ [Apply AOE Damage]
  │                                                       ──(On Cancelled)────────→ [End Ability]
  └─ [Enable Targeting Cursor]
```

The targeting class (e.g., a custom `UTargetingTask`) validates targets and fires completion when confirmed.

### Task Completion and Cleanup

When a task completes, its output pin fires. If the ability ends while a task is still active, the task is automatically canceled:

```
Blueprint:
[Activate Ability]
  ├─ [Wait Delay (5 seconds)] ──→ [Complete Normally]
  └─ [On Ability Ended] ──→ [All Active Tasks Canceled Automatically]
```

**Important**: Always connect task output pins (success, cancel, timeout) to ensure cleanup paths are defined. Do NOT leave tasks without connections.

In C++, manually call `EndTask()` if you need early cancellation:

```cpp
void UMyAbility::CancelWaitTask()
{
    if (WaitTask)
    {
        WaitTask->EndTask();  // Cleanup task explicitly
    }
}
```

---

## Applying Gameplay Effects from Blueprint

### ApplyGameplayEffectToSelf

Applies an instant or duration-based effect to the owning character:

```
Blueprint:
[Activate Ability]
  ├─ [Apply Gameplay Effect To Self]
  │   ├─ In Gameplay Effect Class: GE_Damage
  │   ├─ In Level: 1
  │   └─ In Effect Context: (set source object if needed)
  └─ [Wait for Effect to Apply]
```

The effect applies **server-side** (if authority). Clients see the result via replicated attributes.

```cpp
// C++ equivalent
FGameplayEffectContextHandle EffectContext = AbilitySystemComponent->MakeEffectContext();
EffectContext.AddSourceObject(this);

FGameplayEffectSpecHandle SpecHandle = AbilitySystemComponent->MakeOutgoingSpec(
    EffectClass,
    GetAbilityLevel(),
    EffectContext
);

AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
```

### ApplyGameplayEffectToTarget

Applies an effect to a different character:

```
Blueprint:
[Activate Ability]
  ├─ [Apply Gameplay Effect To Target]
  │   ├─ In Target: [Target Character]
  │   ├─ In Gameplay Effect Class: GE_Heal
  │   ├─ In Level: 1
  │   └─ [Apply Effect]
  └─ [Check if Target was Healed]
```

**Server-side only**: The caller must have authority to apply effects to another character. Clients cannot apply effects to targets.

```cpp
// C++ pattern
UFUNCTION(Server, Reliable)
void ApplyEffectToTarget_Implementation(ACharacter* TargetCharacter)
{
    UAbilitySystemComponent* TargetASC = TargetCharacter->FindComponentByClass<UAbilitySystemComponent>();
    if (!TargetASC)
        return;
    
    FGameplayEffectContextHandle EffectContext = GetAbilitySystemComponent()->MakeEffectContext();
    EffectContext.AddSourceObject(GetAvatarActorFromActorInfo());
    
    FGameplayEffectSpecHandle SpecHandle = GetAbilitySystemComponent()->MakeOutgoingSpec(
        EffectClass,
        GetAbilityLevel(),
        EffectContext
    );
    
    TargetASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
}
```

---

## Gameplay Cues: Static vs Actor

Gameplay Cues trigger visual and audio effects in response to ability events. Two types handle different use cases.

### GameplayCueNotify_Static

One-shot effects (fire-and-forget):

```
Blueprint Class: GC_HitImpact_Static
├─ Parent Class: AGameplayCueNotify_Static
├─ Event: HandleGameplayCue
│   ├─ [Spawn particle effect at impact location]
│   ├─ [Play impact sound]
│   └─ [Return (effect completes)]
```

Fired by:

```
Blueprint Ability:
[Hit Enemy]
  ├─ [Execute Gameplay Cue] (with tag "GameplayCue.Hit.Impact")
  └─ [Effect executes once, no cleanup needed]
```

**Characteristics**:
- No persistent state
- Spawned as an actor once, then destroyed
- Good for: hit sparks, impact sounds, knockback feedback
- Replicated to all clients automatically

### GameplayCueNotify_Actor

Persistent effects (looping):

```
Blueprint Class: GC_FireAura_Actor
├─ Parent Class: AGameplayCueNotify_Actor
├─ Event: OnActive (Called when cue triggered)
│   └─ [Spawn looping fire VFX]
├─ Event: OnTick (Called while active)
│   └─ [Update particle color based on intensity]
└─ Event: OnRemove (Called when cue ends)
    └─ [Fade out and destroy particles]
```

Fired by a Gameplay Effect (infinite or duration-based):

```
Blueprint Class: GE_FireAura
├─ Modifiers:
│   ├─ [Attribute: DamagePerSecond, Magnitude: 10]
│   └─ [Modifier: Periodic (every 0.5s)]
├─ Effects:
│   └─ [Add Gameplay Cue: "GameplayCue.Aura.Fire"]
│   └─ [Tag: "Character.Status.Burning"]
└─ Duration: 5 seconds
   (Cue spawns as actor at start, updates every tick, destroyed at end)
```

**Characteristics**:
- Spawned as an actor, persists until effect ends
- `OnActive` called once on spawn
- `OnTick` called every frame while active
- `OnRemove` called on cleanup
- Good for: fire aura, shield bubble, buff glow
- Replicated to all clients automatically

### Gameplay Cue Triggering

Cues are triggered by:
1. **Gameplay Effects** (via `Cue Tags` list in effect details)
2. **Explicit blueprint call**: `Execute Gameplay Cue` with tag prefix `"GameplayCue.*"`

The tag must have the `GameplayCue.` prefix:

```
Cue Tag: "GameplayCue.Impact.Flesh"
(Registered in Gameplay Cue Manager)

Corresponding blueprint class directory:
Content/GameplayCues/Impact/
  └─ GC_Impact_Flesh_Static.uasset (or _Actor)
```

---

## In-Game Debugging

### showdebug abilitysystem Command

Displays real-time GAS information overlay:

```
Console: showdebug abilitysystem
(Toggle: press ` [backtick] to open console)
```

Shows:
- **Active Abilities**: Currently active ability specs (input ID, level, time active)
- **Active Gameplay Effects**: All applied effects (source, remaining duration, modifiers)
- **Active Gameplay Tags**: All tags currently on the character
- **Ability System Component Stats**: ASC replication mode, ability count, effect count

### Toggle Debug Categories

In code, enable verbose logging:

```cpp
// In DefaultEngine.ini or runtime console
AbilitySystem.Debug.NextCategory
AbilitySystem.Component.Debug 1

// Runtime
LogAbilitySystem = Verbose
```

Or in the editor console:

```
ToggleDebugCategory AbilitySystem
```

This enables per-line logging for every effect applied, tag added/removed, ability activated/deactivated.

---

## Granting Abilities (Server-Side Only)

Abilities are granted server-side and replicate to clients.

### Basic Pattern

```
Blueprint:
[Possess Pawn Event]
  ├─ [Make Gameplay Ability Spec]
  │   ├─ Ability Class: BP_FireAbility
  │   ├─ Level: 1
  │   ├─ Input ID: 0 (keyboard input)
  │   └─ [Create Spec]
  ├─ [Give Ability] (on Ability System Component)
  │   └─ [Spec]
  └─ [Bind Input to Ability]
```

### With Input Binding

```cpp
// C++ pattern
void AMyCharacter::PostInitializeComponents()
{
    Super::PostInitializeComponents();
    
    if (!IsNetMode(NM_Client))  // Server only
    {
        FGameplayAbilitySpec AbilitySpec(
            UMyFireAbility::StaticClass(),
            1,  // Level
            0   // InputID (corresponds to enhanced input action index)
        );
        AbilitySystemComponent->GiveAbility(AbilitySpec);
    }
}
```

### Fire-and-Forget Abilities

For abilities that activate immediately and don't require player input:

```
Blueprint:
[Give Ability And Activate Once]
  ├─ Ability Class: BP_StartupPassive
  └─ [Ability activates immediately, executes, and ends]
```

---

## Tag Querying in Blueprint

### Single Tag Checks

```
Blueprint:
[Has Matching Gameplay Tag]
  ├─ Target: [Character]
  ├─ Tag: "Character.Status.Stunned"
  └─ Returns: true/false
```

```cpp
// C++ equivalent
bool bIsStunned = AbilitySystemComponent->HasMatchingGameplayTag(
    FGameplayTag::RequestGameplayTag(FName("Character.Status.Stunned"))
);
```

### Multi-Tag Checks

#### HasAllMatchingGameplayTags

Returns true only if **all** tags in the container are present:

```
Blueprint:
[Has All Matching Gameplay Tags]
  ├─ Target: [Character]
  ├─ Tags: ["Character.Status.Stunned", "Character.Status.Immobilized"]
  └─ Returns: true (only if BOTH tags present)
```

```cpp
FGameplayTagContainer RequiredTags;
RequiredTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Character.Status.Stunned")));
RequiredTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Character.Status.Immobilized")));

bool bHasAll = AbilitySystemComponent->HasAllMatchingGameplayTags(RequiredTags);
```

#### HasAnyMatchingGameplayTags

Returns true if **any** tag in the container is present:

```
Blueprint:
[Has Any Matching Gameplay Tags]
  ├─ Target: [Character]
  ├─ Tags: ["Character.Status.Stunned", "Character.Status.Knockdown", "Character.Status.Frozen"]
  └─ Returns: true (if ANY of the three tags present)
```

```cpp
FGameplayTagContainer StatusTags;
StatusTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Character.Status.Stunned")));
StatusTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Character.Status.Knockdown")));
StatusTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Character.Status.Frozen")));

bool bHasAny = AbilitySystemComponent->HasAnyMatchingGameplayTags(StatusTags);
```

### Tag Containers vs Individual Tags

**Always use tag containers for multi-tag queries**. Querying individual tags in sequence is slower:

```cpp
// SLOW: Three separate function calls
if (ASC->HasMatchingGameplayTag(Tag1) && 
    ASC->HasMatchingGameplayTag(Tag2) && 
    ASC->HasMatchingGameplayTag(Tag3))
{
    // ...
}

// FAST: One function call with container
FGameplayTagContainer Tags;
Tags.AddTag(Tag1);
Tags.AddTag(Tag2);
Tags.AddTag(Tag3);
if (ASC->HasAllMatchingGameplayTags(Tags))
{
    // ...
}
```

---

## Ability System Module Dependencies

In your project's `.Build.cs`:

```cpp
PublicDependencyModuleNames.AddRange(new string[]
{
    "Core",
    "CoreUObject",
    "Engine",
    "GameplayAbilities",
    "GameplayTags",
    "GameplayTasks"
});
```

Headers:

```cpp
#include "GameplayAbility.h"
#include "GameplayEffect.h"
#include "GameplayAbilitySystem.h"
#include "Abilities/Tasks/AbilityTask_WaitGameplayEvent.h"
#include "Abilities/Tasks/AbilityTask_WaitGameplayTagAdded.h"
#include "Abilities/Tasks/AbilityTask_WaitGameplayTagRemoved.h"
#include "Abilities/Tasks/AbilityTask_WaitDelay.h"
#include "Abilities/Tasks/AbilityTask_WaitTargetData.h"
#include "GameplayCueManager.h"
#include "GameplayCueNotify_Static.h"
#include "GameplayCueNotify_Actor.h"
```

---

## Common Pitfalls

| Pitfall | Issue | Fix |
|---------|-------|-----|
| Ability granted from client | Replication race; server doesn't know | Always grant on server (PostInitializeComponents, RPC) |
| WaitTask without output connections | Task runs but cleanup not guaranteed | Connect all task output pins (success, cancel, timeout) |
| Gameplay Effect applied from client | Server doesn't apply; client sees inconsistency | Wrap in `Server` RPC before effect apply |
| Cue with wrong tag prefix | "Impact.Hit" instead of "GameplayCue.Impact.Hit" | Use "GameplayCue." prefix always |
| Static Cue spawned repeatedly | Memory leak if not cleaned up | Use `_Static` for one-shot, `_Actor` for persistent |
| Tag query with loop | N function calls per tag | Use `HasAllMatchingGameplayTags()` with container instead |

---

## Summary

- **C++ vs Blueprint**: Blueprint for simple logic; C++ for complex conditions and performance-critical abilities
- **Task nodes**: Async operations (`WaitGameplayEvent`, `WaitDelay`, `WaitTargetData`) that auto-cleanup on ability end
- **Effects**: `ApplyGameplayEffectToSelf` (instant/duration), `ApplyGameplayEffectToTarget` (server-only)
- **Cues**: `GameplayCueNotify_Static` (one-shot), `GameplayCueNotify_Actor` (persistent with lifetime)
- **Tag queries**: Use containers (`HasAllMatchingGameplayTags`) instead of individual checks for performance
- **Server-only**: Granting, effect application, task cleanup — all authority-side
- **Debugging**: `showdebug abilitysystem` shows active abilities, effects, tags in real-time

