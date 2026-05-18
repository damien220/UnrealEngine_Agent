# GAS Replication & Dual Init Path — UE5 Reference

## The Core Problem

`UAbilitySystemComponent` requires `InitAbilityActorInfo(OwnerActor, AvatarActor)` before any ability activation or attribute access. In networked games, possession arrives at different times on the server versus the client. Missing either call causes all ability activations to silently fail on the affected machine.

---

## Dual Init Path — Mandatory

```cpp
// AMyCharacter.h
virtual void PossessedBy(AController* NewController) override;  // Server: fires on possession
virtual void OnRep_PlayerState() override;                       // Client: fires when PS replicates

// AMyCharacter.cpp
void AMyCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);
    // Server path — PlayerState exists synchronously
    if (AMyPlayerState* PS = GetPlayerState<AMyPlayerState>())
    {
        AbilitySystemComponent = PS->GetAbilitySystemComponent();
        AbilitySystemComponent->InitAbilityActorInfo(PS, this); // Owner=PS, Avatar=Character
        AttributeSet = AbilitySystemComponent->GetSet<UMyAttributeSet>();
        GrantDefaultAbilities();
    }
}

void AMyCharacter::OnRep_PlayerState()
{
    Super::OnRep_PlayerState();
    // Client path — PlayerState arrives via replication, not synchronously
    if (AMyPlayerState* PS = GetPlayerState<AMyPlayerState>())
    {
        AbilitySystemComponent = PS->GetAbilitySystemComponent();
        AbilitySystemComponent->InitAbilityActorInfo(PS, this);
        BindAbilityInput(); // Re-bind input actions to ability slots
    }
}
```

If ASC lives on `ACharacter` rather than `APlayerState`, init is simpler — call in `BeginPlay` with `HasAuthority()` guard for server-side setup and without for client-side. But this means attributes reset on character respawn.

---

## ASC Replication Modes

Set in `APlayerState` (or `ACharacter`) constructor when creating the ASC:

```cpp
AbilitySystemComponent->SetIsReplicated(true);
AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);
```

| Mode | Full GE data sent to | Minimal sent to | Use case |
|---|---|---|---|
| `Minimal` | Nobody | All | AI actors, single-player |
| `Mixed` | Owning client | Simulated proxies | Player characters (most common) |
| `Full` | All clients | Nobody | When all clients need full GE state (rare) |

`Mixed` is correct for player characters in competitive multiplayer. Simulated proxies receive enough to update cosmetic state; the owning client gets full Gameplay Effect data for ability feedback and prediction.

---

## Ability Prediction — NetExecutionPolicy

```cpp
UGA_Sprint::UGA_Sprint()
{
    // LocalPredicted: client activates immediately, server confirms or rolls back
    // Use for: movement, cosmetic abilities — feels instant to the player
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;

    // ServerInitiated: server activates, result replicates to client — adds one RTT of latency
    // Use for: abilities requiring authoritative validation (purchases, irreversible state changes)
    // NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::ServerInitiated;

    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
}
```

For `LocalPredicted` abilities, GAS automatically creates a `FPredictionKey` scope. All Gameplay Effects and tags applied within `ActivateAbility` are prediction-scoped — the server confirms or rolls back the client's predicted changes.

---

## FPredictionKey — Manual Prediction Scope

When prediction is needed outside a GAS ability (e.g., in a custom movement update), create a prediction scope manually:

```cpp
// On owning client — open a prediction scope
FScopedPredictionWindow ScopedPrediction(AbilitySystemComponent.Get(), true);

FGameplayEffectSpecHandle SpecHandle = AbilitySystemComponent->MakeOutgoingSpec(
    SpeedBoostEffect, 1.f, AbilitySystemComponent->MakeEffectContext());

// Applied immediately on client (predicted), confirmed by server on RPC arrival
AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
```

---

## FGameplayEffectContext — Custom Data Through the Pipeline

Extend `FGameplayEffectContext` to carry hit results, source weapon, or custom payload data:

```cpp
USTRUCT()
struct FMyGameplayEffectContext : public FGameplayEffectContext
{
    GENERATED_BODY()

    FHitResult HitResult;
    TWeakObjectPtr<AActor> SourceWeapon;
    float KnockbackMagnitude = 0.f;

    virtual UScriptStruct* GetScriptStruct() const override
    { return FMyGameplayEffectContext::StaticStruct(); }

    virtual bool NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess) override;
};
```

Register via an `UAbilitySystemGlobals` subclass in `InitGlobalData()`. Without registration, GAS will not serialize or replicate your custom context.

---

## Never Replicate GAS State Manually

Manual `UPROPERTY(Replicated)` on ability state is a GAS integration error — it bypasses prediction, replication ordering, and rollback:

```cpp
// WRONG — bypasses prediction and rollback
UPROPERTY(Replicated)
bool bIsSprinting;

// CORRECT — GAS replicates state via tags on the ASC
AbilitySystemComponent->AddLooseGameplayTag(Tag_Sprinting);

// React on clients via tag change delegates
AbilitySystemComponent->RegisterGameplayTagEvent(Tag_Sprinting, EGameplayTagEventType::NewOrRemoved)
    .AddUObject(this, &AMyCharacter::OnSprintTagChanged);
```

GAS handles replication of: Gameplay Effect specs, attribute values (`DOREPLIFETIME_CONDITION_NOTIFY`), ability specs (granted abilities), and loose tags. Any manual replication of these signals a broken integration that will desync under latency.

---

## Granting Abilities — Server Side Only

```cpp
void AMyCharacter::GrantDefaultAbilities()
{
    if (!HasAuthority() || !IsValid(AbilitySystemComponent)) return;

    for (TSubclassOf<UGameplayAbility> AbilityClass : DefaultAbilities)
    {
        FGameplayAbilitySpec Spec(AbilityClass, 1, INDEX_NONE, this);
        AbilitySystemComponent->GiveAbility(Spec);
    }
}
```

Grant abilities in `PossessedBy` (server path of dual init) — never in `BeginPlay` without an `HasAuthority()` guard.

---

## Debugging GAS Replication

```
showdebug abilitysystem      — overlay showing active abilities, tags, and attributes
AbilitySystem.Debug.NextTarget — cycle debug target to a specific actor
ga.ShowAbilityLogs 1          — log ability activation/cancellation to output
```

After joining as a client, verify: attribute values match the server, granted abilities appear in the debug overlay, and tags applied by the server appear on the client within one RTT.
