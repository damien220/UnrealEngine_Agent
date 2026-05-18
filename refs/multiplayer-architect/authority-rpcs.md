# Authority Model & Server RPCs — UE5 Reference

## The Fundamental Rule

The server is the single source of truth. Clients never directly mutate gameplay state — they send requests (Server RPCs), the server validates them, and the server replicates the authoritative result back to all clients. Clients may predict locally for responsiveness, but prediction is always subject to server correction.

---

## `HasAuthority()` — Where to Put State Changes

Every gameplay state mutation must be guarded by `HasAuthority()`:

```cpp
// WRONG — runs on both server and client; clients corrupt their local state
void AMyActor::TakeDamage(float Amount)
{
    Health -= Amount;
    if (Health <= 0.f) Die();
}

// CORRECT — state mutation only on authority
void AMyActor::TakeDamage(float Amount)
{
    if (!HasAuthority()) return;
    Health -= Amount;
    if (Health <= 0.f) Die();
}
```

`GetLocalRole() == ROLE_Authority` is equivalent but verbose. Always use `HasAuthority()`.

---

## UFUNCTION Specifier Taxonomy

| Specifier | Execution target | Who calls it | Use case |
|---|---|---|---|
| `Server` | Runs on server | Owning client | Client-initiated gameplay request |
| `Client` | Runs on owning client | Server | Server pushes private info to one client |
| `NetMulticast` | Server + all clients | Server | Cosmetic effects all clients should see |

```cpp
// Server RPC — WithValidation mandatory for game-affecting RPCs
UFUNCTION(Server, Reliable, WithValidation)
void ServerFireWeapon(FVector AimDirection, FVector MuzzleLocation);
bool ServerFireWeapon_Validate(FVector AimDirection, FVector MuzzleLocation);
void ServerFireWeapon_Implementation(FVector AimDirection, FVector MuzzleLocation);

// Client RPC — server pushes private notification to one client
UFUNCTION(Client, Reliable)
void ClientShowKillFeed(APlayerState* Killer, APlayerState* Victim);
void ClientShowKillFeed_Implementation(APlayerState* Killer, APlayerState* Victim);

// NetMulticast — cosmetic effect seen by everyone
UFUNCTION(NetMulticast, Unreliable)
void MulticastSpawnImpactFX(FVector Location, FRotator Normal);
void MulticastSpawnImpactFX_Implementation(FVector Location, FRotator Normal);
```

---

## Reliable vs Unreliable

| Choice | Guaranteed delivery | Ordered | Bandwidth cost | When to use |
|---|---|---|---|---|
| `Reliable` | Yes | Yes (in-order) | High (retransmit on loss) | Gameplay-critical: damage, ability activation, item pickup |
| `Unreliable` | No | No | Low | Cosmetic: hit FX, voice, high-frequency position hints |

**Never use `Reliable` for per-frame calls.** A reliable RPC that fires every tick saturates the reliable channel and causes the connection to drop. Create a separate unreliable update path for frequent data:

```cpp
// WRONG — reliable + per-frame = connection saturation
void AMyCharacter::Tick(float DeltaTime)
{
    ServerUpdateAimDirection(GetControlRotation()); // Reliable, every frame
}

// CORRECT — unreliable for frequent updates
UFUNCTION(Server, Unreliable)
void ServerUpdateAimDirection(FRotator AimDir);
```

---

## RPC Ordering Guarantees

Reliable RPCs on the **same actor** arrive in send order. Reliable RPCs on **different actors** have no ordering guarantee relative to each other. For sequences that must be ordered:

- Bundle the data into a single RPC, or
- Use a server-side state machine where each step validates before the next proceeds

---

## Calling Server RPCs Correctly

A `Server` RPC can only be called by the owning client. Calling one from a non-owned actor (AI pawn, world actor) silently drops it:

```cpp
// Guard: only call Server RPCs from the owning local client
if (IsLocallyControlled())
{
    ServerFireWeapon(AimDirection, MuzzleLocation);
}
```

For `Client` RPCs, the actor must be owned by a specific `APlayerController` — the RPC fires only on that connection.
