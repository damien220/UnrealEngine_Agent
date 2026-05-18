# Dedicated Server Build & Configuration — UE5 Reference

## Dedicated vs Listen Server

| Type | Hosts game | Has local player | Use case |
|---|---|---|---|
| Dedicated server | Yes | No — headless process | Competitive multiplayer, ranked modes |
| Listen server | Yes | Yes — host also plays | Co-op, casual, dev testing |

Dedicated servers run `IsRunningDedicatedServer()` == `true`. Use this to skip cosmetic initialization (audio, UI, post-process volumes) that wastes memory and CPU on a headless process.

---

## Server Target File

Every project requires a separate server build target:

```csharp
// Source/MyGameServer.Target.cs
public class MyGameServerTarget : TargetRules
{
    public MyGameServerTarget(TargetInfo Target) : base(Target)
    {
        Type = TargetType.Server;
        DefaultBuildSettings = BuildSettingsVersion.V4;
        IncludeOrderVersion = EngineIncludeOrderVersion.Unreal5_3;
        ExtraModuleNames.Add("MyGame");
        bUseLoggingInShipping = true; // Retain server logs in Shipping
    }
}
```

---

## Packaging — RunUAT Build Command

```bat
:: Package a Linux dedicated server (standard for cloud hosting)
RunUAT.bat BuildCookRun ^
  -project="C:/MyProject/MyGame.uproject" ^
  -platform=Linux ^
  -server ^
  -serverconfig=Shipping ^
  -cook ^
  -build ^
  -stage ^
  -archive ^
  -archivedirectory="C:/MyProject/Build/Server"
```

| Flag | Effect |
|---|---|
| `-platform=Linux` | Target Linux x64 — standard for dedicated server hosting |
| `-server` | Include the server build target |
| `-serverconfig=Shipping` | Shipping config — removes debug checks, optimized binary |
| `-cook` | Cook content for target platform |
| `-stage` / `-archive` | Copy to a deployable directory |

For Windows dedicated servers (dev/test), replace `-platform=Linux` with `-platform=Win64`.

---

## DefaultGame.ini — Server Configuration

```ini
[/Script/EngineSettings.GameMapsSettings]
GameDefaultMap=/Game/Maps/MainMenu
ServerDefaultMap=/Game/Maps/GameLevel_P

[/Script/Engine.GameNetworkManager]
; Total server bandwidth budget (bytes/sec)
TotalNetBandwidth=32000
; Per-connection limits
MaxDynamicBandwidth=7000
MinDynamicBandwidth=4000

[/Script/Engine.Engine]
; Disable frame rate smoothing on server — server runs at fixed tick rate
bSmoothFrameRate=false
```

---

## Skipping Client-Only Init on Server

```cpp
void AMyCharacter::BeginPlay()
{
    Super::BeginPlay();

    if (!IsRunningDedicatedServer())
    {
        // Cosmetic-only: safe to skip on headless server
        InitializeAudioComponent();
        SetupPlayerHUDReferences();
        LoadCharacterSkinAssets();
    }

    // Always run: GAS init, replication setup, server gameplay logic
    InitAbilitySystem();
    RegisterWithSubsystems();
}

void AMyGameMode::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
    Super::InitGame(MapName, Options, ErrorMessage);
    // Parse URL options for session configuration
    MaxPlayers = UGameplayStatics::GetIntOption(Options, TEXT("MaxPlayers"), 10);
    bFriendlyFire = UGameplayStatics::GetIntOption(Options, TEXT("FriendlyFire"), 0) != 0;
}
```

---

## Runtime Bandwidth Configuration

```cpp
void AMyGameMode::PostLogin(APlayerController* NewPlayer)
{
    Super::PostLogin(NewPlayer);

    if (UNetDriver* Driver = GetWorld()->GetNetDriver())
    {
        // Scale per-connection bandwidth with player count
        int32 PlayerCount = FMath::Max(1, GetNumPlayers());
        Driver->MaxClientRate = FMath::Clamp(32000 / PlayerCount, 3000, 7000);
        Driver->MaxInternetClientRate = Driver->MaxClientRate;
    }
}
```

---

## Server Profiling Commands

```
stat net                  — per-frame bytes sent/received, packet count, packet loss
stat netdetailed          — per-actor bandwidth: find which actors consume the most
net.PacketLoss 10         — simulate 10% packet loss for resilience testing
net.PktLag 100            — simulate 100ms one-way latency (200ms RTT)
net.PktLagVariance 20     — simulate ±20ms jitter
```

Always profile at **maximum expected player count** on **actual server hardware** — development machine numbers are not representative. Target: CPU < 30% during peak combat.

---

## Graceful Shutdown

```cpp
void AMyGameMode::RequestServerShutdown(const FText& Reason)
{
    // Notify all connected clients before shutting down
    for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
    {
        if (AMyPlayerController* PC = Cast<AMyPlayerController>(It->Get()))
        {
            PC->ClientReturnToMainMenuWithTextReason(Reason);
        }
    }

    // Allow one frame for notifications to flush, then exit
    FTimerHandle ShutdownTimer;
    GetWorldTimerManager().SetTimer(ShutdownTimer, []()
    {
        FPlatformMisc::RequestExit(false);
    }, 0.5f, false);
}
```

---

## Online Beacon — Lightweight Pre-Session Queries

Use `AOnlineBeaconHostObject` for server-browser pings and player count queries without requiring a full game connection:

```cpp
UCLASS()
class MYGAME_API AMyServerBeaconHostObject : public AOnlineBeaconHostObject
{
    GENERATED_BODY()
public:
    AMyServerBeaconHostObject();

    // Server pushes info to the querying client
    UFUNCTION(Client, Reliable)
    void ClientReceiveServerInfo(int32 CurrentPlayers, int32 MaxPlayers, float AveragePing);
    void ClientReceiveServerInfo_Implementation(int32 CurrentPlayers, int32 MaxPlayers, float AveragePing);
};
```

Beacons are ideal for matchmaking handshakes and server-browser population — they avoid the overhead of a full game session connection just to read lobby info.
