# Replication Graph & Network Prediction Plugin — UE5 Reference

## Replication Graph — When to Enable

The default replication system evaluates relevancy for every actor against every connection every frame — O(actors × connections). At scale this becomes the CPU bottleneck before bandwidth does.

Enable Replication Graph when:
- Server supports > 64 simultaneous connections, or
- Replicated actor count exceeds ~500, or
- `stat net` shows replication evaluation (not bandwidth) as the bottleneck

---

## Enabling Replication Graph

```csharp
// MyGame.Build.cs
PublicDependencyModuleNames.Add("ReplicationGraph");
```

```ini
; DefaultEngine.ini — replace the net driver
[/Script/Engine.GameEngine]
!NetDriverDefinitions=ClearArray
+NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="/Script/ReplicationGraph.ReplicationGraphNetDriver",DriverClassNameFallback="/Script/OnlineSubsystemUtils.IpNetDriver")
```

---

## Custom ReplicationGraph Subclass

```cpp
UCLASS()
class MYGAME_API UMyReplicationGraph : public UReplicationGraph
{
    GENERATED_BODY()

public:
    virtual void InitGlobalActorClassSettings() override;
    virtual void InitGlobalGraphNodes() override;
    virtual void InitConnectionGraphNodes(UNetReplicationGraphConnection* ConnectionManager) override;
    virtual void RouteAddNetworkActorToNodes(
        const FNewReplicatedActorInfo& ActorInfo,
        FGlobalActorReplicationInfo& GlobalInfo) override;
    virtual void RouteRemoveNetworkActorToNodes(
        const FNewReplicatedActorInfo& ActorInfo) override;

private:
    UPROPERTY()
    TObjectPtr<UReplicationGraphNode_GridSpatialization2D> GridNode;

    UPROPERTY()
    TObjectPtr<UReplicationGraphNode_ActorList> AlwaysRelevantNode;
};
```

---

## Spatial Grid Node — Open-World Relevancy

Only replicate actors to connections within spatial proximity of their grid cell:

```cpp
void UMyReplicationGraph::InitGlobalGraphNodes()
{
    // Spatial grid: actors replicate only to nearby connections
    GridNode = CreateNewNode<UReplicationGraphNode_GridSpatialization2D>();
    GridNode->CellSize = 10000.f;                              // 100m cells (UE units = cm)
    GridNode->SpatialBias = FVector2D(-250000.f, -250000.f);   // World min corner
    AddGlobalGraphNode(GridNode);

    // Always-relevant: GameState, PlayerStates — replicate to everyone
    AlwaysRelevantNode = CreateNewNode<UReplicationGraphNode_ActorList>();
    AddGlobalGraphNode(AlwaysRelevantNode);
}

void UMyReplicationGraph::RouteAddNetworkActorToNodes(
    const FNewReplicatedActorInfo& ActorInfo, FGlobalActorReplicationInfo& GlobalInfo)
{
    AActor* Actor = ActorInfo.Actor;

    // GameState and PlayerStates must reach all clients
    if (Actor->IsA<AGameStateBase>() || Actor->IsA<APlayerState>())
    {
        AlwaysRelevantNode->NotifyAddNetworkActor(ActorInfo);
        return;
    }
    // Everything else routes through spatial grid
    GridNode->NotifyAddNetworkActor(ActorInfo);
}
```

---

## Frequency Bucket Node — Staggered Dormant NPC Updates

Instead of sending all dormant NPCs at the same tick, distribute them across update buckets:

```cpp
UCLASS()
class UMyDormantNPCNode : public UReplicationGraphNode_ActorListFrequencyBuckets
{
    GENERATED_BODY()

public:
    UMyDormantNPCNode()
    {
        Settings.NumBuckets = 5;
        // Actors closer to players get higher-frequency buckets automatically
        Settings.BucketThresholds = {2.f, 5.f, 10.f, 20.f}; // distance thresholds (×100cm)
        Settings.DefaultUpdateFrequencyForBucket = {0.5f, 1.f, 2.f, 5.f, 10.f}; // Hz per bucket
    }
};
```

---

## Replication Graph Profiling

```
net.RepGraph.PrintAllNodes       — dump the full node tree and actor counts
net.RepGraph.PrintRouting        — show which node each replicated actor routes to
net.RepGraph.Visualize 1         — enable PIE visualization overlay (spatial grid visible)
stat replicationgraph            — frame timing for RepGraph evaluation
```

Compare Network Profiler bandwidth before and after enabling Replication Graph at maximum player count. In open-world games with 200+ actors, expect 30–60% reduction in replication CPU time.

---

## Network Prediction Plugin (NPP)

NPP provides a rollback-and-reconcile framework for deterministic simulations where the owning client predicts ahead of server confirmation (movement, vehicles, ability-driven locomotion).

**Status as of UE 5.3:** Marked experimental. Use `UCharacterMovementComponent` for standard character movement — it has production-proven prediction built in. Use NPP only when CMC's prediction model cannot cover the simulation.

### Defining Prediction State Types

```cpp
// Input command — what the player did this frame
struct FMyMoveInputCmd
{
    FVector2D MoveInput;    // Normalized move direction
    bool bJumpPressed;
    float DeltaTime;
};

// Sync state — what gets rolled back and reconciled
struct FMyMoveSyncState
{
    FVector Location;
    FVector Velocity;
    bool bIsGrounded;
};

// Aux state — server-authoritative data that does not roll back
struct FMyMoveAuxState
{
    float AirTime;  // Accumulated air time for fall damage calculation
};
```

### Simulation Callback

```cpp
struct FMyMovementSimulation
{
    // Called every simulation tick on server and predicting client — must be deterministic
    static void Tick(
        float DeltaTime,
        const FMyMoveInputCmd& Cmd,
        const FMyMoveSyncState& InputState,
        FMyMoveSyncState& OutputState)
    {
        FVector Accel = FVector(Cmd.MoveInput.X, Cmd.MoveInput.Y, 0.f) * 800.f;
        OutputState.Velocity = InputState.Velocity + Accel * DeltaTime;
        OutputState.Velocity = OutputState.Velocity.GetClampedToMaxSize(MaxSpeed);
        OutputState.Location  = InputState.Location + OutputState.Velocity * DeltaTime;
        OutputState.bIsGrounded = DetectGrounded(OutputState.Location);
    }
};
```

---

## NPP vs UCharacterMovementComponent

| | UCharacterMovementComponent | Network Prediction Plugin |
|---|---|---|
| Setup complexity | Low — subclass and override | High — custom state structs, simulation callback |
| Flexibility | Standard bipedal movement | Any deterministic system |
| Prediction | Built-in, production-proven | Built-in via rollback framework |
| UE5 maturity | Stable | Experimental (5.3+) |
| When to use | Standard character locomotion | Custom physics, vehicles, ability-driven flight |

---

## Custom ReplicationGraph + NPP: Integration Note

When both are active, ensure your Replication Graph routes NPP-managed actors correctly — they have additional requirements on connection-level graph nodes. Register them in `InitConnectionGraphNodes` with a per-connection `UReplicationGraphNode_AlwaysRelevant_ForConnection` node so prediction state reaches the owning client at full frequency regardless of spatial grid cell assignment.
