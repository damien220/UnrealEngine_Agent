# Anti-Cheat & RPC Validation — UE5 Reference

## The Threat Model

Every client calling a Server RPC is a potential attacker sending arbitrary inputs. `_Validate` executes on the server **before** `_Implementation`. If `_Validate` returns `false`, UE5 immediately disconnects the client. Use this aggressively — validation that passes impossible inputs is a cheat vector.

---

## _Validate Pattern — Mandatory on All Game-Affecting RPCs

```cpp
// AMyCharacter.h
UFUNCTION(Server, Reliable, WithValidation)
void ServerRequestFireWeapon(FVector AimDirection, FVector MuzzleLocation);
bool ServerRequestFireWeapon_Validate(FVector AimDirection, FVector MuzzleLocation);
void ServerRequestFireWeapon_Implementation(FVector AimDirection, FVector MuzzleLocation);

// AMyCharacter.cpp
bool AMyCharacter::ServerRequestFireWeapon_Validate(FVector AimDirection, FVector MuzzleLocation)
{
    // Reject NaN/Inf — crash physics and trace calculations
    if (AimDirection.ContainsNaN() || MuzzleLocation.ContainsNaN()) return false;
    if (!AimDirection.IsNormalized()) return false;

    // Reject muzzle location impossibly far from the character (wall-hack muzzle teleport)
    if (FVector::Dist(GetActorLocation(), MuzzleLocation) > 250.f) return false;

    // Reject if weapon is not equipped or not in a state that allows firing
    if (!IsValid(CurrentWeapon) || !CurrentWeapon->CanFire()) return false;

    return true;
}
```

---

## Common Cheat Vectors and Validation Patterns

### Distance Exploits
```cpp
bool AMyCharacter::ServerRequestPickupItem_Validate(AActor* Item)
{
    if (!IsValid(Item)) return false;
    // Client claims they picked up this item — verify proximity
    return FVector::Dist(GetActorLocation(), Item->GetActorLocation()) <= 300.f;
}
```

### Rate Exploits (RPC Flooding)
```cpp
// Server-side per-ability rate tracking (not replicated)
TMap<int32, float> AbilityLastUsedTime;

bool AMyCharacter::ServerActivateAbility_Validate(int32 AbilityID)
{
    float Now = GetWorld()->GetTimeSeconds();
    float& LastUse = AbilityLastUsedTime.FindOrAdd(AbilityID, 0.f);
    const float MinCooldown = 0.1f; // No ability fires faster than 100ms
    if (Now - LastUse < MinCooldown) return false;
    LastUse = Now;
    return true;
}
```

### Value Range Exploits
```cpp
bool AMyCharacter::ServerSetMovementMultiplier_Validate(float Multiplier)
{
    // Reject values outside the range any legitimate ability could produce
    return Multiplier >= 0.f && Multiplier <= MaxLegitimateSpeedMultiplier;
}
```

### Self-Targeting and Cross-Player Exploits
```cpp
bool AMyCharacter::ServerDealDamage_Validate(AActor* Target, float Amount)
{
    if (!IsValid(Target)) return false;
    if (Target == this) return false;             // No self-damage score exploit
    if (Amount <= 0.f) return false;              // No healing-via-damage exploit
    if (Amount > MaxSingleHitDamage) return false; // No one-shot god-mode exploit
    return true;
}
```

---

## Re-Check in _Implementation

`_Validate` runs immediately on RPC receipt. `_Implementation` runs after the RPC is dispatched — state may have changed in between. Always re-verify preconditions:

```cpp
void AMyCharacter::ServerRequestFireWeapon_Implementation(
    FVector AimDirection, FVector MuzzleLocation)
{
    // State may have changed between Validate and Implementation
    if (!IsValid(CurrentWeapon)) return;
    if (CurrentWeapon->GetCurrentAmmo() <= 0) return;
    if (!CanFire()) return; // Stunned, dead, reloading, mid-reload

    CurrentWeapon->ConsumeAmmo(1);
    FiringComponent->ApplyHitscan(MuzzleLocation, AimDirection);
}
```

---

## Audit Logging — Suspicious Input Detection

Log anomalous inputs before disconnecting — high-latency clients can produce false positives on distance checks. Log first, escalate on repeated violations:

```cpp
void AMyCharacter::LogSuspiciousRPC(const FString& RPCName, const FString& Reason)
{
    if (APlayerState* PS = GetPlayerState())
    {
        UE_LOG(LogAntiCheat, Warning,
            TEXT("Suspicious | Player: %s | RPC: %s | Reason: %s | Pos: %s"),
            *PS->GetUniqueId().ToString(), *RPCName, *Reason,
            *GetActorLocation().ToString());
    }
}

bool AMyCharacter::ServerTeleportToSpawn_Validate(FVector DestLocation)
{
    float Dist = FVector::Dist(GetActorLocation(), DestLocation);
    if (Dist > 1000.f)
    {
        LogSuspiciousRPC(TEXT("ServerTeleportToSpawn"),
            FString::Printf(TEXT("Distance %.0f exceeds max 1000"), Dist));
        return false;
    }
    return true;
}
```

---

## Security Audit Checklist

Run before every milestone and before every public build:

- [ ] Every `UFUNCTION(Server, ...)` that affects gameplay state has `WithValidation`
- [ ] No `_Validate` returns `true` unconditionally — every function has real checks
- [ ] All numeric inputs are range-validated: no NaN, no negative damage, no out-of-bounds IDs
- [ ] All `AActor*` and `UObject*` inputs are `IsValid()`-checked before dereference
- [ ] Distance checks exist for every RPC where spatial proximity matters (pickup, melee, interact)
- [ ] Rate limiting exists for every ability or action that has a cooldown
- [ ] `_Implementation` re-checks preconditions after `_Validate`
- [ ] No gameplay-critical state is mutated outside an `HasAuthority()` guard
- [ ] Verify: can a client directly trigger another player's damage, score change, or item pickup? (Answer must be: No)
- [ ] Verify: can a client send values that would crash the server (NaN, divide-by-zero inputs)? (Answer must be: No)

---

## EAC & BattlEye Plugin Integration

UE5 ships EAC and BattlEye stubs. Enable via `.uproject` and `DefaultEngine.ini`.

### .uproject plugin entries
```json
{
    "Plugins": [
        { "Name": "EasyAntiCheat", "Enabled": true },
        { "Name": "BattlEye",      "Enabled": false }
    ]
}
```

### DefaultEngine.ini
```ini
[/Script/Engine.Engine]
DefaultPlatformService=EOS

[OnlineSubsystemEOS]
bEnabled=true
bEnableEAC=true
bEnableBattlEye=false   ; Set true if using BattlEye instead
```

**Flow**: On server startup the EOS SDK initializes the anti-cheat service. The client connects, anti-cheat verifies binary integrity, and the session is only established if the check passes. Server-side EAC functions are `EOS_AntiCheatServer_*` — accessed via the EOS SDK, not UE's native API.

**Mobile / console**: EAC is PC/console only. Do not enable on mobile targets — the plugin is a no-op and adds unnecessary build complexity.

---

## Obfuscating Server-Authoritative Values

Never replicate raw authoritative values (exact health, ammo) to non-owning clients — they are visible in memory and packet captures.

### Owner-only replication
```cpp
// Exact values — only the owning client needs them
UPROPERTY(Replicated) float Health;
UPROPERTY(Replicated) int32 Ammo;

// Coarse indicator — all clients see this for UI/status bars
UENUM(BlueprintType)
enum class EHealthLevel : uint8 { Full, High, Medium, Low, Critical };
UPROPERTY(ReplicatedUsing=OnRep_HealthLevel) EHealthLevel HealthLevel;

void AMyCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME_CONDITION(AMyCharacter, Health,      COND_OwnerOnly);
    DOREPLIFETIME_CONDITION(AMyCharacter, Ammo,        COND_OwnerOnly);
    DOREPLIFETIME(AMyCharacter, HealthLevel);  // Enum visible to all — no exploitable precision
}
```

### Updating the coarse indicator server-side
```cpp
void AMyCharacter::Server_TakeDamage_Implementation(float DamageAmount)
{
    Health = FMath::Max(0.f, Health - DamageAmount);

    if      (Health > 80.f) HealthLevel = EHealthLevel::Full;
    else if (Health > 50.f) HealthLevel = EHealthLevel::High;
    else if (Health > 20.f) HealthLevel = EHealthLevel::Medium;
    else if (Health >  0.f) HealthLevel = EHealthLevel::Low;
    else                    HealthLevel = EHealthLevel::Critical;
}
```

**Rule**: Apply `COND_OwnerOnly` to health, ammo, currency, inventory counts, and any value that gives a gameplay advantage when known. Replicate only a quantized or enum-level representation to all other clients.
