# Sessions & Matchmaking

**Applies to**: UE5.0+ | **Module**: `OnlineSubsystem`, `OnlineSubsystemUtils` | **Header**: `OnlineSubsystem.h`, `OnlineSessionSettings.h`, `Interfaces/OnlineSessionInterface.h`

---

## Core Session Lifecycle

The `IOnlineSessionInterface` manages sessions through four async operations, each firing a delegate when complete. **Do NOT chain calls** — always wait for the delegate before proceeding.

### Create Session

```cpp
// Header
void AMyGameMode::CreateGameSession()
{
    IOnlineSubsystem* OnlineSub = IOnlineSubsystem::Get();
    if (!OnlineSub) return;
    
    IOnlineSessionPtr SessionInterface = OnlineSub->GetSessionInterface();
    if (!SessionInterface.IsValid()) return;
    
    // Cache the delegate handle for later cleanup
    FOnCreateSessionCompleteDelegate Delegate = FOnCreateSessionCompleteDelegate::CreateUObject(
        this, &AMyGameMode::OnCreateSessionComplete);
    OnCreateSessionCompleteDelegateHandle = 
        SessionInterface->AddOnCreateSessionCompleteDelegate_Handle(Delegate);
    
    FOnlineSessionSettings SessionSettings;
    SessionSettings.NumPublicConnections = 4;
    SessionSettings.bIsLANMatch = false;  // true for LAN testing with Null subsystem
    SessionSettings.bUsesPresence = true;  // Steam/EOS presence
    SessionSettings.bAllowJoinInProgress = true;
    SessionSettings.bShouldAdvertise = true;
    
    // Add custom key-value data
    SessionSettings.Set(SETTING_MAPNAME, FString(TEXT("MyMap")));
    
    // CreateSession params: HostingPlayerNum, SessionName, Settings
    SessionInterface->CreateSession(0, NAME_GameSession, SessionSettings);
}

void AMyGameMode::OnCreateSessionComplete(FName SessionName, bool bWasSuccessful)
{
    IOnlineSubsystem* OnlineSub = IOnlineSubsystem::Get();
    if (!OnlineSub) return;
    
    IOnlineSessionPtr SessionInterface = OnlineSub->GetSessionInterface();
    if (SessionInterface.IsValid() && bWasSuccessful)
    {
        // Session created; now ServerTravel to the game map
        GetWorld()->ServerTravel(TEXT("/Game/Maps/GameplayMap?listen"));
    }
    else
    {
        UE_LOG(LogGame, Error, TEXT("Failed to create session"));
    }
}
```

**Timing**: Session creation is async and can take 50–300ms depending on the subsystem (faster for Null/LAN, slower for Steam/EOS).

---

### Find Sessions

```cpp
void AMyMainMenuGameMode::FindGameSessions()
{
    IOnlineSubsystem* OnlineSub = IOnlineSubsystem::Get();
    if (!OnlineSub) return;
    
    IOnlineSessionPtr SessionInterface = OnlineSub->GetSessionInterface();
    if (!SessionInterface.IsValid()) return;
    
    // Create search parameters
    SessionSearch = MakeShareable(new FOnlineSessionSearch());
    SessionSearch->bIsLanQuery = false;  // true for LAN (Null subsystem)
    SessionSearch->MaxSearchResults = 50;
    
    // Optional: set custom query filters (key-value pairs from SessionSettings)
    SessionSearch->QuerySettings.Set(SEARCH_PRESENCE, 1, EOnlineComparisonOp::Equals);
    
    FOnFindSessionsCompleteDelegate Delegate = 
        FOnFindSessionsCompleteDelegate::CreateUObject(
            this, &AMyMainMenuGameMode::OnFindSessionsComplete);
    OnFindSessionsCompleteDelegateHandle = 
        SessionInterface->AddOnFindSessionsCompleteDelegate_Handle(Delegate);
    
    // Find sessions for the local player (index 0)
    SessionInterface->FindSessions(0, SessionSearch.ToSharedRef());
}

void AMyMainMenuGameMode::OnFindSessionsComplete(bool bWasSuccessful)
{
    IOnlineSubsystem* OnlineSub = IOnlineSubsystem::Get();
    if (!OnlineSub) return;
    
    IOnlineSessionPtr SessionInterface = OnlineSub->GetSessionInterface();
    if (!SessionInterface.IsValid()) return;
    
    // Remove delegate handle to avoid memory leak
    SessionInterface->ClearOnFindSessionsCompleteDelegate_Handle(
        OnFindSessionsCompleteDelegateHandle);
    
    if (!bWasSuccessful || !SessionSearch.IsValid())
    {
        UE_LOG(LogGame, Warning, TEXT("Session search failed"));
        return;
    }
    
    // Access results from SessionSearch->SearchResults (TArray<FOnlineSessionSearchResult>)
    for (const FOnlineSessionSearchResult& Result : SessionSearch->SearchResults)
    {
        FString SessionName = Result.Session.OwningUserName;
        int32 MaxPlayers = Result.Session.SessionSettings.NumPublicConnections;
        FString MapName;
        Result.Session.SessionSettings.Get(SETTING_MAPNAME, MapName);
        
        UE_LOG(LogGame, Warning, TEXT("Found session: %s, Max: %d, Map: %s"),
               *SessionName, MaxPlayers, *MapName);
    }
}
```

**Return Value**: `OnFindSessionsComplete` receives a bool indicating success. Results are **not** passed as a parameter; read `SessionSearch->SearchResults` after the delegate fires.

---

### Join Session

```cpp
void AMyMainMenuGameMode::JoinGameSession(const FOnlineSessionSearchResult& SelectedResult)
{
    IOnlineSubsystem* OnlineSub = IOnlineSubsystem::Get();
    if (!OnlineSub) return;
    
    IOnlineSessionPtr SessionInterface = OnlineSub->GetSessionInterface();
    if (!SessionInterface.IsValid()) return;
    
    FOnJoinSessionCompleteDelegate Delegate = 
        FOnJoinSessionCompleteDelegate::CreateUObject(
            this, &AMyMainMenuGameMode::OnJoinSessionComplete);
    OnJoinSessionCompleteDelegateHandle = 
        SessionInterface->AddOnJoinSessionCompleteDelegate_Handle(Delegate);
    
    // JoinSession params: LocalPlayerNum, SessionName, SessionToJoin
    SessionInterface->JoinSession(0, NAME_GameSession, SelectedResult);
}

void AMyMainMenuGameMode::OnJoinSessionComplete(FName SessionName, 
                                                 EOnJoinSessionCompleteResult::Type Result)
{
    IOnlineSubsystem* OnlineSub = IOnlineSubsystem::Get();
    if (!OnlineSub) return;
    
    IOnlineSessionPtr SessionInterface = OnlineSub->GetSessionInterface();
    if (!SessionInterface.IsValid()) return;
    
    SessionInterface->ClearOnJoinSessionCompleteDelegate_Handle(
        OnJoinSessionCompleteDelegateHandle);
    
    if (Result == EOnJoinSessionCompleteResult::Success)
    {
        // Get the connection string (server address)
        FString ConnectString;
        if (SessionInterface->GetResolvedConnectString(SessionName, ConnectString))
        {
            // Travel to server
            if (APlayerController* PC = GetWorld()->GetFirstPlayerController())
            {
                PC->ClientTravel(ConnectString, TRAVEL_Absolute);
            }
        }
    }
    else
    {
        UE_LOG(LogGame, Error, TEXT("Failed to join session: %d"), (int32)Result);
    }
}
```

**Connection Path**: On successful join, call `GetResolvedConnectString()` to obtain the server address, then `ClientTravel()` to connect.

---

### Destroy Session

```cpp
void AMyGameMode::EndGameSession()
{
    IOnlineSubsystem* OnlineSub = IOnlineSubsystem::Get();
    if (!OnlineSub) return;
    
    IOnlineSessionPtr SessionInterface = OnlineSub->GetSessionInterface();
    if (!SessionInterface.IsValid()) return;
    
    FOnDestroySessionCompleteDelegate Delegate = 
        FOnDestroySessionCompleteDelegate::CreateUObject(
            this, &AMyGameMode::OnDestroySessionComplete);
    OnDestroySessionCompleteDelegateHandle = 
        SessionInterface->AddOnDestroySessionCompleteDelegate_Handle(Delegate);
    
    SessionInterface->DestroySession(NAME_GameSession);
}

void AMyGameMode::OnDestroySessionComplete(FName SessionName, bool bWasSuccessful)
{
    IOnlineSubsystem* OnlineSub = IOnlineSubsystem::Get();
    if (!OnlineSub) return;
    
    IOnlineSessionPtr SessionInterface = OnlineSub->GetSessionInterface();
    if (SessionInterface.IsValid())
    {
        SessionInterface->ClearOnDestroySessionCompleteDelegate_Handle(
            OnDestroySessionCompleteDelegateHandle);
    }
    
    if (bWasSuccessful)
    {
        UE_LOG(LogGame, Warning, TEXT("Session destroyed"));
        // Return to main menu, etc.
    }
}
```

---

## FOnlineSessionSettings Reference

Core session configuration object — passed to `CreateSession()` and `FindSessions()`.

| Property | Type | Purpose |
|----------|------|---------|
| `NumPublicConnections` | `int32` | Max players on the session (visible in search). Default: 0. |
| `NumPrivateConnections` | `int32` | Reserved slots (not visible in search). |
| `bIsLANMatch` | `bool` | **true** for LAN/Null (PIE testing), **false** for Steam/EOS. |
| `bUsesPresence` | `bool` | **true** to enable Steam/EOS presence and automatic friend invites. |
| `bAllowJoinInProgress` | `bool` | **true** to allow joining mid-game. |
| `bShouldAdvertise` | `bool` | **true** to broadcast to matchmaking backends. |
| `bIsDedicated` | `bool` | **true** for dedicated server sessions. |
| Custom KV pairs | (see below) | Set via `Set(KEY, VALUE)` method. |

### Custom Data

```cpp
SessionSettings.Set(SETTING_MAPNAME, FString(TEXT("MyMap")));
SessionSettings.Set(SETTING_GAMEMODE, FString(TEXT("TDM")));
SessionSettings.Set(FString(TEXT("MyCustomKey")), FString(TEXT("MyValue")));

// Retrieve in Find callback
FString MapName;
SearchResult.Session.SessionSettings.Get(SETTING_MAPNAME, MapName);
```

**Built-in Keys** (defined in `OnlineSessionSettings.h`):
- `SETTING_MAPNAME` — Level to load
- `SETTING_GAMEMODE` — Game type
- `SETTING_PLAYTIME` — Session playtime limit
- `SETTING_NUMBOTS` — AI count

---

## Online Subsystem Configuration

### DefaultEngine.ini

```ini
[/Script/Engine.Engine]
; Default platform service (Null for testing, Steam for production)
DefaultPlatformService=Null

[OnlineSubsystem]
PollingIntervalInMs=0
MaxAuthRetries=5
bHasVoiceEnabled=false

[OnlineSubsystemNull]
bEnabled=true

[OnlineSubsystemSteam]
bEnabled=true
SteamDevAppId=480
bVACEnabled=true
bAllowP2PPacketRelay=true

[/Script/Engine.GameEngine]
; Define net drivers for session-based travel
+NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="OnlineSubsystemSteam.SteamNetDriver",DriverClassNameFallback="OnlineSubsystemUtils.IpNetDriver")
+NetDriverDefinitions=(DefName="LoginNetDriver",DriverClassName="OnlineSubsystemUtils.IpNetDriver")
```

**Platform Selection**:
- **Null subsystem**: For LAN testing (PIE, couch co-op). Does not require authentication.
- **Steam subsystem**: Requires `SteamDevAppId` (get from Steamworks partner dashboard). Provides presence, matchmaking, P2P networking.
- **EOS subsystem**: Requires EOS SDK and credential setup (beyond scope of this doc).

---

## Module Dependencies

In `YourGame.Build.cs`:

```csharp
PublicDependencyModuleNames.AddRange(new string[]
{
    "Core",
    "CoreUObject",
    "Engine",
    "OnlineSubsystem",        // Session interface
    "OnlineSubsystemUtils",   // Helper functions with World context
    "OnlineSubsystemNull",    // LAN/PIE testing (if enabled)
    "OnlineSubsystemSteam",   // Steam backend (if enabled)
});
```

**Why `OnlineSubsystemUtils`?** Functions in Utils take the current World into account, avoiding multi-world issues in PIE.

---

## Server Travel with Sessions

Sessions persist across `ServerTravel()` (server-initiated map change). Use this pattern:

```cpp
// In GameMode on session creation
void AMyGameMode::BeginPlay()
{
    Super::BeginPlay();
    
    // Create session, then travel
    GetWorld()->ServerTravel(TEXT("/Game/Maps/GameplayMap?listen"));
    // The "?listen" parameter tells Unreal the server is listening for clients
}
```

**ServerTravel vs ClientTravel**:
- `ServerTravel` — Server-side only, all clients follow (session persists).
- `ClientTravel` — Client initiates connection (disconnects from old session first).

---

## Seamless Travel (Advanced)

For large maps or long load screens, enable seamless travel to avoid showing a black screen.

### DefaultEngine.ini

```ini
[/Script/Engine.GameMapsSettings]
TransitionMap=/Game/Maps/TransitionMap
; TransitionMap is a minimal, fast-loading level (50MB or less)
```

### GameMode C++

```cpp
void AMyGameMode::BeginPlay()
{
    Super::BeginPlay();
    bUseSeamlessTravel = true;  // Enable seamless travel
}

// Later, initiate transition
void AMyGameMode::StartSeamlessTravel()
{
    // Travel to transition map, then to final map
    GetWorld()->ServerTravel(TEXT("/Game/Maps/TransitionMap?game=/Game/Maps/FinalMap"));
}
```

**How It Works**: Server travels to a minimal TransitionMap while the final map loads in the background. No session interruption; all players remain connected. Load time is masked by the transition.

**Pitfall**: TransitionMap must exist and be fast-loading (< 1 second). Omitting it forces a hard cutover.

---

## Known Pitfalls

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Delegate not firing | Delegate handle not stored; garbage collected | Store `FDelegateHandle` in class member |
| Session visible after destroy | Async delay; API state != backend state | Wait 2–5 seconds before creating new session |
| FindSessions returns 0 results | `bIsLanQuery` mismatch; session not advertised | Ensure `bShouldAdvertise=true`; check subsystem config |
| Join fails silently | Session full; player already in session | Check `NumPublicConnections` > 0; call DestroySession first |
| GetResolvedConnectString returns empty | Session not created locally; joining from search | Call this **after** JoinSession succeeds, not before |

---

## References

- [IOnlineSession API (UE5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/OnlineSubsystem/Interfaces/IOnlineSession/)
- [FOnlineSessionSettings (UE5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/OnlineSubsystem/FOnlineSessionSettings)
- [Session Management Blog (Cedric Neukirchen)](https://cedric-neukirchen.net/docs/session-management/)
- [Online Subsystem (UE5.7 Docs)](https://dev.epicgames.com/documentation/en-us/unreal-engine/online-subsystem-in-unreal-engine)
