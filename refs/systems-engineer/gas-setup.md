# Gameplay Ability System (GAS) — Setup & Requirements Reference

## What GAS Is

GAS is Unreal's built-in framework for data-driven, network-replicated gameplay abilities, attributes, and effects. It handles replication, prediction, cooldowns, costs, and tag-based filtering out of the box. It is the standard architecture for ability systems in shipping UE5 titles.

---

## Module Setup (.Build.cs)

GAS requires three module dependencies. Missing any one of them causes compile errors across GAS headers.

```csharp
public class MyGame : ModuleRules
{
    public MyGame(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

        PublicDependencyModuleNames.AddRange(new string[]
        {
            "Core", "CoreUObject", "Engine", "InputCore",
            "GameplayAbilities",   // GAS core: UAbilitySystemComponent, UGameplayAbility, UAttributeSet
            "GameplayTags",        // FGameplayTag, FGameplayTagContainer, tag registry
            "GameplayTasks"        // UGameplayTask — async task infrastructure used by GAS internally
        });

        PrivateDependencyModuleNames.AddRange(new string[]
        {
            "Slate", "SlateCore"
        });
    }
}
```

After modifying `.Build.cs`, run `GenerateProjectFiles.bat` and rebuild. If IntelliSense shows GAS headers as missing, close the IDE and regenerate.

---

## UAbilitySystemComponent

Every actor that participates in GAS must have a `UAbilitySystemComponent` (ASC). For characters this is typically added in C++ on the `ACharacter` subclass or on a dedicated `APlayerState`.

```cpp
// In ACharacter subclass
UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "GAS")
TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;

// In constructor
AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
AbilitySystemComponent->SetIsReplicated(true);
AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);
// Mixed = server sends GEs to owning client, minimal for simulated proxies
```

### Replication Modes

| Mode | Who gets Gameplay Effects | Use case |
|---|---|---|
| `Minimal` | Owner only | Single-player or AI |
| `Mixed` | Full to owner, minimal to others | Player characters (most common) |
| `Full` | Full GE data to all clients | When all clients need full GE state (rare) |

---

## UAttributeSet

All game stats (health, stamina, mana, armor) must derive from `UAttributeSet`. Each attribute uses `FGameplayAttributeData` and the `ATTRIBUTE_ACCESSORS` helper macro.

```cpp
UCLASS()
class MYGAME_API UMyAttributeSet : public UAttributeSet
{
    GENERATED_BODY()

public:
    // Health
    UPROPERTY(BlueprintReadOnly, Category = "Attributes", ReplicatedUsing = OnRep_Health)
    FGameplayAttributeData Health;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Health)

    UPROPERTY(BlueprintReadOnly, Category = "Attributes", ReplicatedUsing = OnRep_MaxHealth)
    FGameplayAttributeData MaxHealth;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, MaxHealth)

    // Stamina
    UPROPERTY(BlueprintReadOnly, Category = "Attributes", ReplicatedUsing = OnRep_Stamina)
    FGameplayAttributeData Stamina;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Stamina)

    // Required overrides
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) override;
    virtual void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data) override;

    UFUNCTION()
    void OnRep_Health(const FGameplayAttributeData& OldHealth);

    UFUNCTION()
    void OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth);

    UFUNCTION()
    void OnRep_Stamina(const FGameplayAttributeData& OldStamina);
};
```

```cpp
// .cpp replication setup
void UMyAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Health, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, MaxHealth, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Stamina, COND_None, REPNOTIFY_Always);
}

void UMyAttributeSet::OnRep_Health(const FGameplayAttributeData& OldHealth)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, Health, OldHealth);
}
```

**`PreAttributeChange`**: Use to clamp incoming values before they are applied (e.g., clamp Health to [0, MaxHealth]).
**`PostGameplayEffectExecute`**: Use to react after a Gameplay Effect modifies an attribute (e.g., trigger death when Health reaches 0).

---

## UGameplayAbility

All abilities derive from `UGameplayAbility`. Override `ActivateAbility` and `EndAbility` at minimum.

```cpp
UCLASS()
class MYGAME_API UGA_Sprint : public UGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_Sprint();

    virtual void ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData) override;

    virtual void EndAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        bool bReplicateEndAbility,
        bool bWasCancelled) override;

    virtual bool CanActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayTagContainer* SourceTags = nullptr,
        const FGameplayTagContainer* TargetTags = nullptr,
        OUT FGameplayTagContainer* OptionalRelevantTags = nullptr) const override;

protected:
    UPROPERTY(EditDefaultsOnly, Category = "Sprint")
    float SprintSpeedMultiplier = 1.5f;

    UPROPERTY(EditDefaultsOnly, Category = "Sprint")
    FGameplayTag SprintingTag;  // e.g., "State.Sprinting"
};
```

```cpp
UGA_Sprint::UGA_Sprint()
{
    // Replicate: server runs ability, client gets predicted activation
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
}

void UGA_Sprint::ActivateAbility(...)
{
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // Apply tag to owning actor
    ActorInfo->AbilitySystemComponent->AddLooseGameplayTag(SprintingTag);
    // Apply speed modifier via Gameplay Effect here
}

void UGA_Sprint::EndAbility(...)
{
    ActorInfo->AbilitySystemComponent->RemoveLooseGameplayTag(SprintingTag);
    Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
}
```

---

## FGameplayTag — Mandatory Over Strings

Use `FGameplayTag` for all gameplay event identifiers, state flags, and ability conditions. Never use raw strings or plain enums for GAS-related identity.

```cpp
// WRONG — string-based ability identification
if (AbilityName == FName("Sprint")) { ... }

// CORRECT — tag-based
FGameplayTag SprintTag = FGameplayTag::RequestGameplayTag(FName("Ability.Sprint"));
if (ActorASC->HasMatchingGameplayTag(SprintTag)) { ... }
```

**Tag hierarchy in `DefaultGameplayTags.ini`**:
```ini
[/Script/GameplayTags.GameplayTagsSettings]
GameplayTagList=(Tag="Ability.Sprint",DevComment="Sprint ability active")
GameplayTagList=(Tag="Ability.Attack.Melee",DevComment="Melee attack ability")
GameplayTagList=(Tag="State.Stunned",DevComment="Actor is stunned")
GameplayTagList=(Tag="State.Dead",DevComment="Actor is dead, suppress all abilities")
GameplayTagList=(Tag="Effect.Damage.Fire",DevComment="Fire damage effect type")
```

**Tag containers for blocking and requirements**:
```cpp
// In UGameplayAbility subclass constructor
ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag("State.Stunned"));
ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag("State.Dead"));
ActivationRequiredTags.AddTag(FGameplayTag::RequestGameplayTag("State.Grounded"));
```

---

## Granting Abilities

Abilities must be granted to the ASC before they can be activated. Grant on `BeginPlay` (server-side):

```cpp
void AMyCharacter::BeginPlay()
{
    Super::BeginPlay();

    if (HasAuthority() && IsValid(AbilitySystemComponent))
    {
        for (TSubclassOf<UGameplayAbility> AbilityClass : DefaultAbilities)
        {
            FGameplayAbilitySpec Spec(AbilityClass, 1, INDEX_NONE, this);
            AbilitySystemComponent->GiveAbility(Spec);
        }
    }
}
```

---

## Replication Through UAbilitySystemComponent

**Never replicate ability state manually.** GAS handles:
- `FGameplayEffectSpec` replication (GameplayEffects and their modifiers)
- `FGameplayAbilitySpec` replication (which abilities are granted)
- `FGameplayAttributeData` replication via `DOREPLIFETIME_CONDITION_NOTIFY`
- Tag replication via `AddLooseGameplayTag` (replicated if ASC is replicated)

Manual `UPROPERTY(Replicated)` on ability state is a sign of a GAS integration error. Move that state into an Attribute or a Gameplay Effect.

---

## Initialization Order for PlayerState-Hosted ASC

When ASC lives on `APlayerState` (recommended for persistent player data across respawns):

```cpp
// In APlayerState constructor
AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("ASC"));
AbilitySystemComponent->SetIsReplicated(true);

// In AMyCharacter::PossessedBy (server-side possession)
void AMyCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);
    AMyPlayerState* PS = GetPlayerState<AMyPlayerState>();
    if (IsValid(PS))
    {
        AbilitySystemComponent = PS->GetAbilitySystemComponent();
        AbilitySystemComponent->InitAbilityActorInfo(PS, this); // Owner=PS, Avatar=Character
    }
}

// In AMyCharacter::OnRep_PlayerState (client-side)
void AMyCharacter::OnRep_PlayerState()
{
    Super::OnRep_PlayerState();
    AMyPlayerState* PS = GetPlayerState<AMyPlayerState>();
    if (IsValid(PS))
    {
        AbilitySystemComponent = PS->GetAbilitySystemComponent();
        AbilitySystemComponent->InitAbilityActorInfo(PS, this);
    }
}
```

Failing to call `InitAbilityActorInfo` on both server and client is the most common GAS setup bug — it causes all ability activations to silently fail on the client.
