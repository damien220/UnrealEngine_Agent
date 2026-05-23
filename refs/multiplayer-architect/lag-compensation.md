# Lag Compensation & Server Rewind

**Applies to**: UE5.0+ | **Concepts**: Not built-in; requires custom implementation | **Typical Use**: Hitbox validation, weapon traces, melee attacks in 50+ ms latency environments

---

## Server-Side Rewind Overview

**Definition**: Server rewind stores historical actor transforms and allows the server to "replay" a trace or hit test against the actor's position at the client's shot timestamp, compensating for network latency.

**When to Use**:
- Competitive shooters (weapon hit detection, melee validation)
- High-latency environments (50+ ms ping where instant-server traces feel "unfair")
- Asymmetric authority (server authoritative but clients deserve latency-fair feedback)

**When NOT to Use**:
- RPGs with slow, forgiving hit windows
- Projects where simple server-authoritative traces are "good enough"
- Bandwidth-constrained environments (rewind buffers consume memory)

---

## Transform History Storage

Store actor transforms in a ring buffer, sampled every network tick (~100ms or 10 ticks/sec).

```cpp
// In MyCharacter.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "MyCharacter.generated.h"

struct FServerSideRewindSnapshot
{
    FVector Location;
    FRotator Rotation;
    double Timestamp;  // FDateTime::UtcNow() when captured
};

UCLASS()
class MYGAME_API AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    AMyCharacter();

    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;

protected:
    // Ring buffer: max 300ms of history @ 10Hz = ~3 snapshots
    static constexpr int32 MAX_REWIND_SNAPSHOTS = 30;  // Adjust for your tick rate
    TArray<FServerSideRewindSnapshot> RewindHistory;
    int32 RewindIndex = 0;
    
    double LastRewindSampleTime = 0.0;
    static constexpr double REWIND_SAMPLE_INTERVAL = 0.1;  // 100ms between samples

    // Capture current transform to ring buffer
    void CaptureRewindSnapshot();
    
    // Rewind hitbox to a specific timestamp, perform trace, restore
    bool CheckHitAtTimestamp(double TargetTimestamp, const FVector& TraceStart,
                             const FVector& TraceEnd, FHitResult& OutHit);
};
```

### Implementation

```cpp
// In MyCharacter.cpp
#include "MyCharacter.h"
#include "GameFramework/CharacterMovementComponent.h"

AMyCharacter::AMyCharacter()
{
    PrimaryActorTick.TickInterval = 0.0f;
    PrimaryActorTick.bCanEverTick = true;
}

void AMyCharacter::BeginPlay()
{
    Super::BeginPlay();
    
    if (GetLocalRole() == ROLE_Authority)
    {
        RewindHistory.Reserve(MAX_REWIND_SNAPSHOTS);
    }
}

void AMyCharacter::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    
    // Only server stores history
    if (GetLocalRole() != ROLE_Authority) return;
    
    double Now = FDateTime::UtcNow().ToUnixTimestamp();
    if (Now - LastRewindSampleTime >= REWIND_SAMPLE_INTERVAL)
    {
        CaptureRewindSnapshot();
        LastRewindSampleTime = Now;
    }
}

void AMyCharacter::CaptureRewindSnapshot()
{
    FServerSideRewindSnapshot Snapshot;
    Snapshot.Location = GetActorLocation();
    Snapshot.Rotation = GetActorRotation();
    Snapshot.Timestamp = FDateTime::UtcNow().ToUnixTimestamp();
    
    if (RewindHistory.Num() >= MAX_REWIND_SNAPSHOTS)
    {
        // Ring buffer: overwrite oldest
        RewindHistory[RewindIndex] = Snapshot;
    }
    else
    {
        RewindHistory.Add(Snapshot);
    }
    
    RewindIndex = (RewindIndex + 1) % MAX_REWIND_SNAPSHOTS;
}

bool AMyCharacter::CheckHitAtTimestamp(double TargetTimestamp, const FVector& TraceStart,
                                       const FVector& TraceEnd, FHitResult& OutHit)
{
    // Find closest snapshot to target time
    FServerSideRewindSnapshot* ClosestSnapshot = nullptr;
    double ClosestDiff = FLT_MAX;
    
    for (const FServerSideRewindSnapshot& Snap : RewindHistory)
    {
        double TimeDiff = FMath::Abs(Snap.Timestamp - TargetTimestamp);
        if (TimeDiff < ClosestDiff)
        {
            ClosestDiff = TimeDiff;
            ClosestSnapshot = const_cast<FServerSideRewindSnapshot*>(&Snap);
        }
    }
    
    if (!ClosestSnapshot)
    {
        return false;  // No history available
    }
    
    // Save current transform
    FVector OriginalLocation = GetActorLocation();
    FRotator OriginalRotation = GetActorRotation();
    
    // Temporarily rewind to snapshot
    SetActorLocation(ClosestSnapshot->Location, false, nullptr, ETeleportType::TeleportPhysics);
    SetActorRotation(ClosestSnapshot->Rotation);
    
    // Perform trace against rewound position
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(this);  // Don't hit yourself
    
    GetWorld()->LineTraceSingleByChannel(OutHit, TraceStart, TraceEnd,
                                          CC_Pawn, QueryParams);
    
    // Restore original transform
    SetActorLocation(OriginalLocation, false, nullptr, ETeleportType::TeleportPhysics);
    SetActorRotation(OriginalRotation);
    
    return OutHit.bBlockingHit;
}
```

---

## Client Timestamp & RPC Pattern

Client sends shot timestamp with hit RPC. Server uses it to rewind.

```cpp
// In MyCharacter.h
UCLASS()
class MYGAME_API AMyCharacter : public ACharacter
{
    // ...
    
    // Client fires weapon, sends hit to server
    UFUNCTION(Server, Reliable, WithValidation)
    void Server_RequestHit(const FVector& ShotLocation, const FVector& TraceDirection, 
                           double ClientShotTimestamp);
    
    bool Server_RequestHit_Validate(const FVector& ShotLocation, const FVector& TraceDirection,
                                    double ClientShotTimestamp);
};
```

### Implementation with Timestamp Validation

```cpp
// In MyCharacter.cpp

void AMyCharacter::FireWeapon()
{
    if (!IsLocallyControlled()) return;
    
    FVector ShotStart = GetActorLocation();
    FVector ShotDir = GetActorForwardVector();
    
    // Send to server with client's timestamp
    double ClientTime = GetWorld()->GetTimeSeconds();  // Or custom network time sync
    Server_RequestHit(ShotStart, ShotDir, ClientTime);
}

void AMyCharacter::Server_RequestHit_Implementation(const FVector& ShotLocation,
                                                    const FVector& TraceDirection,
                                                    double ClientShotTimestamp)
{
    // Rewind and check hit
    FVector TraceEnd = ShotLocation + TraceDirection * 10000.0f;
    FHitResult HitResult;
    
    if (CheckHitAtTimestamp(ClientShotTimestamp, ShotLocation, TraceEnd, HitResult))
    {
        if (AMyCharacter* HitCharacter = Cast<AMyCharacter>(HitResult.GetActor()))
        {
            HitCharacter->TakeDamage(50.0f, FDamageEvent(), nullptr, this);
            
            UE_LOG(LogGame, Warning, TEXT("Hit confirmed at timestamp: %.3f"), 
                   ClientShotTimestamp);
        }
    }
}

bool AMyCharacter::Server_RequestHit_Validate(const FVector& ShotLocation,
                                              const FVector& TraceDirection,
                                              double ClientShotTimestamp)
{
    // Validate timestamp is not too far in the past (exploit check)
    double ServerTime = GetWorld()->GetTimeSeconds();
    double TimeDiff = ServerTime - ClientShotTimestamp;
    
    // Allow 500ms of client latency + buffer
    if (TimeDiff < 0.0 || TimeDiff > 0.5)
    {
        UE_LOG(LogGame, Warning, TEXT("Invalid hit timestamp from client: %.3f"), 
               ClientShotTimestamp);
        return false;
    }
    
    // Validate direction is normalized
    if (TraceDirection.IsNormalized() == false || TraceDirection.IsZero())
    {
        return false;
    }
    
    return true;
}
```

---

## Network Time Synchronization

Use `APlayerController::GetServerTime()` for a client-synced, lag-corrected server timestamp.

```cpp
// Client-side: get server time
APlayerController* PC = GetWorld()->GetFirstPlayerController();
if (PC)
{
    float ServerTime = PC->GetServerTime();
    // This is synced with server via replication
    // Clients use this to timestamp their actions
}
```

**How It Works**: Server sends its current time in replication ticks. Client interpolates to account for latency, providing a "what time is it on the server right now?" value.

**Alternative (Custom)**: Roll your own by sending server `FDateTime::UtcNow()` to clients periodically and tracking offset.

---

## SimulatedProxy Interpolation Settings

For networked non-owned characters, UE5's `UCharacterMovementComponent` can smooth their movement between received updates.

```cpp
// In Character constructor or GameMode setup
void AMyCharacter::PostInitializeComponents()
{
    Super::PostInitializeComponents();
    
    if (CharacterMovement)
    {
        // Smoothing mode for simulated proxies
        CharacterMovement->NetworkSmoothingMode = ENetworkSmoothingMode::Linear;
        
        // Don't snap to exact position if update is far (visual smoothing)
        CharacterMovement->NetworkMaxSmoothUpdateDistance = 256.0f;  // Snap if > 256cm
        CharacterMovement->NetworkNoSmoothUpdateDistance = 32.0f;    // Never snap if < 32cm
        
        // Optional: match tick rate to network updates
        CharacterMovement->NetworkSimulatedSmoothRotationTime = 0.1f;  // Rotate over 100ms
    }
}
```

**Config Reference**:
- `NetworkSmoothingMode` — `Linear`, `Exponential`, or `Disabled`
- `NetworkMaxSmoothUpdateDistance` — Threshold to snap vs. smooth (256cm is typical)
- `NetworkNoSmoothUpdateDistance` — Always snap below this threshold (32cm for fine movements)

**Note**: Do NOT manually set `OwnerOnlyReplicatedMovement` or call `SetActorLocation()` on simulated proxies. Character Movement handles this automatically.

---

## AGameNetworkManager Bandwidth

Configure server-side bandwidth limits for movement replication.

### DefaultGame.ini

```ini
[/Script/Engine.GameNetworkManager]
; Bandwidth settings (in bits/second)
TotalNetBandwidth=104857600      ; 100 Mbps total
MaxDynamicBandwidth=104857600
MinDynamicBandwidth=10485760      ; 10 Mbps minimum (fallback)

; Movement packet size estimate
MoveRepSize=512.0                 ; Expected size of ServerMove() RPC in bytes

; Adjust per actor class
; (See UNetDriver config for per-actor NetUpdateFrequency)
ClientAdjustUpdateCost=512.0
```

### Per-Actor Tick Rate

```cpp
// In GameMode or Character class
void AMyCharacter::BeginPlay()
{
    Super::BeginPlay();
    
    // Update position 10 times per second (100ms intervals)
    // Reduce for distant players via GetNetPriority() system
    NetUpdateFrequency = 10.0f;
    MinNetUpdateFrequency = 2.0f;  // Never slower than 2Hz
}
```

**Rule of Thumb**: 
- **High-action (shooters)**: 20–30 Hz per player
- **Medium (action RPG)**: 10–15 Hz
- **Slow (strategy)**: 2–5 Hz

Lower tick rates save bandwidth but degrade visual smoothness; rewind lag compensation helps mitigate.

---

## Server Authority vs. Client-Side Prediction Trade-Offs

| Model | Latency Feel | Anti-Cheat | CPU Cost | Bandwidth |
|-------|--------------|-----------|----------|-----------|
| **Pure server-authoritative** | Laggy (1 RTT delay) | Perfect | Low | Low |
| **Server + rewind** | Fair (near-instant) | Good (with validation) | High | Moderate |
| **Client-predicted + server verify** | Best (instant) | Requires rate-limiting & checks | Moderate | High |

For competitive games, **server + rewind is the sweet spot**: feels fair, maintains authority, scales reasonably.

---

## Known Pitfalls

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Rewind buffer too short | History size < 300ms of latency | Increase `MAX_REWIND_SNAPSHOTS` or reduce `REWIND_SAMPLE_INTERVAL` |
| Hitbox not rewound correctly | Calling check on non-rewound actor | Ensure `CheckHitAtTimestamp()` resets position after trace |
| Timestamp mismatch | Client time != server time | Use `GetServerTime()` or implement clock sync before firing |
| Memory spike with many players | Ring buffer per character × count | Pre-allocate buffers in BeginPlay; consider streaming/cull distant buffers |
| False negatives (hit not detected) | Snapshot timestamp between client request and closest sample | Use linear interpolation between snapshots for sub-sample accuracy |

---

## References

- [SnapNet: Server-Side Rewind (UE5 Blog)](https://snapnet.dev/blog/performing-lag-compensation-in-unreal-engine-5/)
- [SnapNet: Server Rewind Docs](https://snapnet.dev/docs/unreal-engine-sdk/manual/server-rewind/)
- [Understanding Networked Movement (UE5.7 Docs)](https://dev.epicgames.com/documentation/unreal-engine/understanding-networked-movement-in-the-character-movement-component-for-unreal-engine)
- [GitHub: UE5 Server-Side Rewind Demo](https://github.com/marcohenning/ue5-server-side-rewind)
- [FDateTime (UE5.7 API Docs)](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Core/Misc/FDateTime)
