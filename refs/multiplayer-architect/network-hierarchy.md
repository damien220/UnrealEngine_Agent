# Network Object Hierarchy — UE5 Reference

## The Four Layers

| Class | Replication | Scope | Owns |
|---|---|---|---|
| `AGameMode` | Never — server only | Server | Rule arbitration, win conditions, respawn logic |
| `AGameState` | All clients | World | Round state, team scores, shared timers |
| `APlayerState` | All clients | Per player | Public player data: name, ping, kills, team |
| `APlayerController` | Owning client only | Per player | Input, camera, HUD, client-side authority |

Violating this hierarchy causes replication bugs that are difficult to diagnose. Assign state to the correct layer from the start.

---

## AGameMode — Server Only

`AGameMode` is never replicated and does not exist on clients. Use it only for server-authoritative decisions. `GetWorld()->GetAuthGameMode()` returns `nullptr` on clients — always guard.

```cpp
UCLASS()
class MYGAME_API AMyGameMode : public AGameModeBase
{
    GENERATED_BODY()

public:
    virtual void PostLogin(APlayerController* NewPlayer) override;
    virtual void Logout(AController* Exiting) override;

    // Server only — arbitrate win condition, mutate GameState
    void CheckWinCondition();
    void StartRespawnTimer(APlayerController* DeadPlayer, float Delay);

private:
    // These fields exist only on the server — never replicated
    int32 MaxRoundsToWin = 3;
    TMap<APlayerController*, int32> PlayerRoundWins;
};
```

**Common violation:** Storing round-timer or score logic in `GameMode` and trying to display it on clients. Anything clients must display belongs in `AGameState`.

---

## AGameState — World-Wide Shared State

`AGameState` replicates to all connected clients. It is the correct location for any state that all players need to see simultaneously.

```cpp
UCLASS()
class MYGAME_API AMyGameState : public AGameStateBase
{
    GENERATED_BODY()

public:
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    UPROPERTY(Replicated)
    float RoundTimeRemaining = 300.f;

    UPROPERTY(Replicated)
    int32 TeamAScore = 0;

    UPROPERTY(Replicated)
    int32 TeamBScore = 0;

    // Use RepNotify when clients must react to phase transitions
    UPROPERTY(ReplicatedUsing = OnRep_GamePhase)
    EGamePhase CurrentPhase = EGamePhase::Warmup;

    UFUNCTION()
    void OnRep_GamePhase();
};
```

`AGameMode` mutates `AGameState` on the server. `AGameState` replicates automatically to all clients. Clients never call RPCs on `AGameState` directly.

---

## APlayerState — Per-Player Public Data

`APlayerState` replicates to all clients and **persists across character respawns**. `ACharacter` can be destroyed and re-spawned; `APlayerState` survives. Place all public per-player data here.

```cpp
UCLASS()
class MYGAME_API AMyPlayerState : public APlayerState
{
    GENERATED_BODY()

public:
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    UPROPERTY(Replicated) int32 Kills = 0;
    UPROPERTY(Replicated) int32 Deaths = 0;
    UPROPERTY(Replicated) ETeam Team = ETeam::None;

    // ASC on PlayerState — ability data persists across respawns
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;
};
```

**ASC placement decision:**
- **On `APlayerState`**: ability data (attributes, granted abilities) persists across respawns. Correct for competitive multiplayer where progression/loadouts survive death.
- **On `ACharacter`**: ability data resets on death. Simpler but less flexible.

---

## APlayerController — Owning Client Only

`APlayerController` replicates only to its owning client's connection. Use `Client` RPCs to push private information server-to-client. Other clients cannot see or access another player's `APlayerController`.

```cpp
UCLASS()
class MYGAME_API AMyPlayerController : public APlayerController
{
    GENERATED_BODY()

public:
    // Server pushes a private kill notification to this client only
    UFUNCTION(Client, Reliable)
    void ClientReceiveKillNotification(APlayerState* Killer, APlayerState* Victim);
    void ClientReceiveKillNotification_Implementation(APlayerState* Killer, APlayerState* Victim);

    // Server notifies this client that a respawn point is ready
    UFUNCTION(Client, Reliable)
    void ClientOnRespawnReady(FVector SpawnLocation);
    void ClientOnRespawnReady_Implementation(FVector SpawnLocation);
};
```

**Common violation:** Storing team scores or round state in `APlayerController` and expecting other clients to read it. Shared state belongs in `AGameState`.

---

## Hierarchy Access Patterns

```cpp
// From any actor — access the full hierarchy
AMyGameState* GS = GetWorld()->GetGameState<AMyGameState>();

// From ACharacter / AController
AMyPlayerState* PS = GetPlayerState<AMyPlayerState>();

// From ACharacter
AMyPlayerController* PC = Cast<AMyPlayerController>(GetController());

// Server only — returns nullptr on clients
AMyGameMode* GM = GetWorld()->GetAuthGameMode<AMyGameMode>();

// Iterate all players from server
if (IsValid(GS))
{
    for (APlayerState* PlayerState : GS->PlayerArray)
    {
        if (AMyPlayerState* MyPS = Cast<AMyPlayerState>(PlayerState))
        {
            // ...
        }
    }
}
```

---

## Violation Consequence Reference

| Violation | Symptom |
|---|---|
| Gameplay logic in `GameMode`, displayed on clients | Clients never see the state — `GameMode` is server-only |
| Shared score stored in `PlayerController` | Only the owning client sees it |
| Private data placed in `GameState` | All clients see data they should not |
| ASC on `Character` when cross-respawn persistence needed | Attributes reset on death |
| `GetAuthGameMode()` called on client without null guard | Null dereference crash |
