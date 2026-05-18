# Replication Efficiency — UE5 Reference

## DOREPLIFETIME Macros

All replicated properties must be registered in `GetLifetimeReplicatedProps`. A property declared `UPROPERTY(Replicated)` but missing from this function never replicates.

```cpp
void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // Unconditional — all clients receive every update
    DOREPLIFETIME(AMyActor, TeamScore);

    // Owner only — private state, only the owning connection receives it
    DOREPLIFETIME_CONDITION(AMyActor, PrivateAmmo, COND_OwnerOnly);

    // Simulated proxies only — skip the owning client (they predict locally)
    DOREPLIFETIME_CONDITION(AMyActor, SimulatedPosition, COND_SimulatedOnly);

    // Initial only — sent once on spawn, never again
    DOREPLIFETIME_CONDITION(AMyActor, SpawnConfig, COND_InitialOnly);

    // Skip owner — everyone except the owning client (owner predicts it)
    DOREPLIFETIME_CONDITION(AMyActor, AimRotation, COND_SkipOwner);
}
```

### Condition Reference

| Condition | Who receives updates | When to use |
|---|---|---|
| `COND_None` | All clients | Shared world state (health, position) |
| `COND_OwnerOnly` | Owning client only | Inventory, ammo count, private stats |
| `COND_SimulatedOnly` | Non-owning clients | Cosmetic movement the owner predicts |
| `COND_SkipOwner` | All clients except owner | State the owner predicts locally |
| `COND_InitialOnly` | All clients, once on spawn | Immutable spawn config (team, character class) |
| `COND_ReplayOrOwner` | Owner + replay system | Replay-critical private data |

---

## NetUpdateFrequency — Per-Actor Tuning

The default replication rate is higher than most actors need. Set explicitly in constructors:

```cpp
// Projectile — fast-moving, accuracy critical
AMyProjectile::AMyProjectile()
{
    bReplicates = true;
    NetUpdateFrequency = 100.f;
    MinNetUpdateFrequency = 33.f;
}

// Player character — interpolated on clients, 20Hz is smooth
AMyCharacter::AMyCharacter()
{
    bReplicates = true;
    NetUpdateFrequency = 20.f;
    MinNetUpdateFrequency = 5.f;
}

// NPC enemy — interpolated, 10Hz acceptable
AMyNPCEnemy::AMyNPCEnemy()
{
    bReplicates = true;
    NetUpdateFrequency = 10.f;
    MinNetUpdateFrequency = 2.f;
}

// Environment actor (door, switch) — state changes infrequently
AMyEnvironmentActor::AMyEnvironmentActor()
{
    bReplicates = true;
    NetUpdateFrequency = 2.f;
    MinNetUpdateFrequency = 0.f; // Only send on change
}
```

---

## Network Priority — Dynamic Replication Ordering

When bandwidth is constrained, UE5 drops lower-priority actor updates. Override `GetNetPriority` to boost actors closest to or in front of the viewer:

```cpp
float AMyActor::GetNetPriority(
    const FVector& ViewPos, const FVector& ViewDir,
    AActor* Viewer, AActor* ViewTarget,
    UActorChannel* InChannel, float Time, bool bLowBandwidth) const
{
    float Priority = Super::GetNetPriority(ViewPos, ViewDir, Viewer, ViewTarget, InChannel, Time, bLowBandwidth);

    float Distance = FVector::Dist(ViewPos, GetActorLocation());
    if (Distance < 500.f) Priority *= 2.f;

    // Boost for actors inside the viewer's ~45° forward cone
    FVector ToActor = (GetActorLocation() - ViewPos).GetSafeNormal();
    if (FVector::DotProduct(ViewDir, ToActor) > 0.7f) Priority *= 1.5f;

    return Priority;
}
```

---

## Actor Dormancy — Zero-Cost Idle Actors

Dormant actors send zero replication updates until explicitly woken. Use for world actors whose state changes infrequently:

```cpp
// After the actor reaches a stable state, go dormant
void AMyChest::OnOpened()
{
    bIsOpen = true;
    FlushNetDormancy(); // Force one final update so all clients get current state
    SetNetDormancy(DORM_DormantAll);
}

// Wake before any subsequent state change
void AMyChest::ResetForNextRound()
{
    SetNetDormancy(DORM_Awake);
    bIsOpen = false;
}
```

`FlushNetDormancy()` is critical — it ensures all clients receive the terminal state before updates stop.

---

## RepNotify — Client Reactions to State Changes

Use `ReplicatedUsing` when clients need to react to a property change (UI update, sound, animation):

```cpp
UPROPERTY(ReplicatedUsing = OnRep_Health)
float Health = 100.f;

UFUNCTION()
void AMyCharacter::OnRep_Health()
{
    HealthBarWidget->SetPercent(Health / MaxHealth);
    if (Health <= 0.f) PlayDeathMontage();
}
```

The previous value is not automatically available in RepNotify. Cache it as a separate member if you need the delta:

```cpp
float PreviousHealth = 0.f; // not replicated — only needed locally

UFUNCTION()
void AMyCharacter::OnRep_Health()
{
    bool bTookDamage = Health < PreviousHealth;
    PreviousHealth = Health;
    if (bTookDamage) PlayHurtSound();
}
```

For `UAttributeSet` attributes, use `DOREPLIFETIME_CONDITION_NOTIFY` with `REPNOTIFY_Always` — GAS attributes must notify even when the value does not change numerically (e.g., restored to max triggers a UI update).

---

## Bandwidth Profiling

```
stat net              — per-frame bytes sent/received, packet loss
stat netdetailed      — per-actor bandwidth breakdown
net.PacketLoss 10     — simulate 10% packet loss
net.PktLag 100        — simulate 100ms one-way latency
```

Target: **< 15KB/s per player** at maximum player count, measured with Network Profiler on actual dedicated server hardware.
