# Voice Chat & Proximity Audio

**Applies to**: UE5.0+ (modern VoiceChat plugin) | **Module**: `VoiceChat`, `OnlineSubsystem` | **Header**: `Voice/VoiceModule.h`, `Interfaces/OnlineVoiceInterface.h`

---

## Voice Chat System Architecture

UE5 provides a **unified Voice Chat Interface** (`IVoiceChat`) that abstracts multiple backends (EOS, Vivox, platform-specific VOIP). One `IVoiceChatUser` instance exists per local player; multiple channels can be joined per user.

---

## Enabling Built-In Voice

### DefaultEngine.ini

```ini
[Voice]
; Enable the voice module globally
bEnabled=true

[OnlineSubsystem]
; Notify online subsystem that voice is available
bHasVoiceEnabled=true
```

### DefaultGame.ini

```ini
[/Script/Engine.GameSession]
; Require push-to-talk (if using PTT, set to true)
bRequiresPushToTalk=false

[VoiceChat]
; EOS-specific voice config
bUseEOS=true
```

---

## VoiceChat Plugin Setup (EOS/Vivox Backend)

### YourGame.Build.cs

```csharp
PublicDependencyModuleNames.AddRange(new string[]
{
    "Core",
    "CoreUObject",
    "Engine",
    "OnlineSubsystem",
    "OnlineSubsystemEOS",  // or OnlineSubsystemVivox
    "VoiceChat",           // Modern voice chat interface
});
```

### YourGame.uproject

```json
{
    "Plugins": [
        {
            "Name": "VoiceChat",
            "Enabled": true
        },
        {
            "Name": "OnlineSubsystemEOS",
            "Enabled": true
        }
    ]
}
```

---

## Voice Chat Workflow (Pseudocode Pattern)

```cpp
// In PlayerController or GameState

#include "Voice/VoiceModule.h"
#include "Voice/IVoiceChat.h"

void AMyPlayerController::SetupVoice()
{
    IVoiceChat* VoiceChat = IVoiceChat::Get();
    if (!VoiceChat)
    {
        UE_LOG(LogGame, Error, TEXT("VoiceChat module not available"));
        return;
    }
    
    // Step 1: Login with credentials
    // EOS uses your EOS User ID; Vivox uses Vivox account token
    FVoiceChatLoginCredentials Creds;
    Creds.UserName = GetPlayerState()->GetPlayerName();
    // (More setup depends on backend; see backend-specific docs)
    
    VoiceChat->Login(Creds, [this](const FVoiceChatLoginResult& Result)
    {
        if (Result.IsSuccess)
        {
            UE_LOG(LogGame, Warning, TEXT("VoiceChat login successful"));
            OnVoiceChatLoginComplete();
        }
        else
        {
            UE_LOG(LogGame, Error, TEXT("VoiceChat login failed: %s"), 
                   *Result.ErrorMessage);
        }
    });
}

void AMyPlayerController::OnVoiceChatLoginComplete()
{
    IVoiceChat* VoiceChat = IVoiceChat::Get();
    if (!VoiceChat) return;
    
    // Step 2: Join a voice channel (positional or non-positional)
    FVoiceChatChannel GameChannel = FVoiceChatChannel::CreateGameChannel();
    GameChannel.Channel = FString(TEXT("game_channel"));
    
    // For spatial audio, set 3D parameters
    GameChannel.bPositional = true;
    GameChannel.MaxDistance = 2700.0f;  // 27 meters (default in Vivox)
    GameChannel.MinDistance = 90.0f;    // 0.9 meters for full volume
    
    VoiceChat->JoinChannel(GameChannel, [this](const FVoiceChatChannel& Channel,
                                                const FVoiceChatResult& Result)
    {
        if (Result.IsSuccess)
        {
            UE_LOG(LogGame, Warning, TEXT("Joined voice channel: %s"), *Channel.Channel);
        }
        else
        {
            UE_LOG(LogGame, Error, TEXT("Failed to join channel: %s"), 
                   *Result.ErrorMessage);
        }
    });
}

// Update voice positions every frame (for spatial channels)
void AMyPlayerController::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    
    IVoiceChat* VoiceChat = IVoiceChat::Get();
    if (!VoiceChat) return;
    
    if (ACharacter* PlayerChar = GetCharacter())
    {
        // Update listener position (where sounds come from)
        FVector ListenerPos = PlayerChar->GetActorLocation();
        FVector ListenerForward = PlayerChar->GetActorForwardVector();
        FVector ListenerUp = PlayerChar->GetActorUpVector();
        
        VoiceChat->Set3DPosition(ListenerPos, ListenerForward, ListenerUp);
    }
    
    // Update other players' positions via replicated actor positions
    // (The voice subsystem reads replicated positions from ACharacter/APawn directly)
}

// Mute/unmute specific players
void AMyPlayerController::MutePlayer(const FString& PlayerName, bool bMute)
{
    IVoiceChat* VoiceChat = IVoiceChat::Get();
    if (!VoiceChat) return;
    
    if (bMute)
    {
        VoiceChat->MutePlayer(PlayerName);
    }
    else
    {
        VoiceChat->UnmutePlayer(PlayerName);
    }
}

// Set per-player volume
void AMyPlayerController::SetPlayerVolume(const FString& PlayerName, float Volume)
{
    IVoiceChat* VoiceChat = IVoiceChat::Get();
    if (!VoiceChat) return;
    
    // Volume range: 0.0 (silent) to 1.0 (full)
    VoiceChat->SetAudioVolume(PlayerName, FMath::Clamp(Volume, 0.0f, 1.0f));
}

// Leave channel (on disconnect)
void AMyPlayerController::LeaveVoice()
{
    IVoiceChat* VoiceChat = IVoiceChat::Get();
    if (!VoiceChat) return;
    
    VoiceChat->LeaveChannel(TEXT("game_channel"), [](const FVoiceChatResult& Result)
    {
        if (Result.IsSuccess)
        {
            UE_LOG(LogGame, Warning, TEXT("Left voice channel"));
        }
    });
}
```

---

## Push-to-Talk (PTT) with Enhanced Input

Use Enhanced Input System to bind a key to voice transmission.

### Setup UInputAction

In Editor:
1. Create a new `UInputAction` (Content Browser → Input → Input Action)
2. Name it `IA_PushToTalk`
3. Assign a key (e.g., Spacebar, V, or a gamepad button)

### PlayerController Implementation

```cpp
#include "EnhancedInputComponent.h"
#include "EnhancedInputSubsystems.h"
#include "InputActionValue.h"
#include "Voice/VoiceModule.h"

void AMyPlayerController::BeginPlay()
{
    Super::BeginPlay();
    
    // Setup Enhanced Input
    if (UEnhancedInputComponent* EnhancedInputComponent = 
        FindComponentByClass<UEnhancedInputComponent>())
    {
        if (UInputAction* PushToTalkAction = LoadObject<UInputAction>(
            nullptr, TEXT("/Script/EnhancedInput.InputAction'/Game/Input/IA_PushToTalk.IA_PushToTalk'")))
        {
            // Bind Started (key pressed) → start transmission
            EnhancedInputComponent->BindAction(PushToTalkAction, 
                ETriggerEvent::Started, this, &AMyPlayerController::OnPushToTalkStarted);
            
            // Bind Completed (key released) → stop transmission
            EnhancedInputComponent->BindAction(PushToTalkAction, 
                ETriggerEvent::Completed, this, &AMyPlayerController::OnPushToTalkCompleted);
        }
    }
}

void AMyPlayerController::OnPushToTalkStarted()
{
    IVoiceChat* VoiceChat = IVoiceChat::Get();
    if (!VoiceChat) return;
    
    // Enable transmission (server-side or client-side, depending on architecture)
    VoiceChat->SetTransmitChannel(TEXT("game_channel"));
    
    // Optional: broadcast to other players that this player is talking
    Server_SetTalking(true);
}

void AMyPlayerController::OnPushToTalkCompleted()
{
    IVoiceChat* VoiceChat = IVoiceChat::Get();
    if (!VoiceChat) return;
    
    // Disable transmission
    VoiceChat->SetTransmitChannel(TEXT(""));
    
    Server_SetTalking(false);
}

UFUNCTION(Server, Reliable, WithValidation)
void AMyPlayerController::Server_SetTalking(bool bIsTalking)
{
    if (APlayerState* PS = GetPlayerState<APlayerState>())
    {
        // Replicate talking state for other players' UI
        PS->bIsTalking = bIsTalking;  // Custom replicated property
    }
}

bool AMyPlayerController::Server_SetTalking_Validate(bool bIsTalking)
{
    // Prevent spam (rate-limit PTT toggle)
    static double LastToggleTime = 0.0;
    double Now = FPlatformTime::Seconds();
    if (Now - LastToggleTime < 0.1)  // Min 100ms between toggles
    {
        return false;
    }
    LastToggleTime = Now;
    return true;
}
```

---

## Legacy UVOIPTalker Component (Deprecated but Still Supported)

For projects using the older voice system, `UVOIPTalker` is an `UActorComponent` attached to `APlayerState`.

```cpp
// In PlayerState constructor or BeginPlay
void AMyPlayerState::BeginPlay()
{
    Super::BeginPlay();
    
    // Create VOIP talker (legacy)
    UVOIPTalker* VoipTalker = NewObject<UVOIPTalker>(this);
    VoipTalker->RegisterComponent();
    
    // Settings for proximity audio
    VoipTalker->VoiceSettings.MaxHearDistance = 2700.0f;  // 27m
    VoipTalker->VoiceSettings.MaxTalkDistance = 4000.0f;  // 40m (broadcast range)
}
```

**Status**: This component is superseded by `IVoiceChat`. Use only if your project requires legacy compatibility.

---

## Proximity Chat (Spatial Audio)

Modern voice (EOS/Vivox) handles spatialization automatically if:
1. Channel is created with `bPositional = true`
2. **All players' character positions are replicated** on the network
3. Listener position updated via `IVoiceChat::Set3DPosition()` each frame

### Attenuation Model

Vivox default 3D properties (applies to EOS as well):
- **Min distance**: 0.9m → full volume
- **Max distance**: 27m → silent
- **Falloff curve**: linear attenuation between min and max

### Custom Attenuation

```cpp
void AMyPlayerController::SetProximityDistance(float MaxDistance)
{
    IVoiceChat* VoiceChat = IVoiceChat::Get();
    if (!VoiceChat) return;
    
    // Leave current channel
    VoiceChat->LeaveChannel(TEXT("game_channel"), [this, MaxDistance](const FVoiceChatResult& R)
    {
        if (R.IsSuccess)
        {
            // Rejoin with new distance
            FVoiceChatChannel GameChannel = FVoiceChatChannel::CreateGameChannel();
            GameChannel.Channel = TEXT("game_channel");
            GameChannel.bPositional = true;
            GameChannel.MaxDistance = MaxDistance;
            GameChannel.MinDistance = 0.9f;
            
            IVoiceChat::Get()->JoinChannel(GameChannel, [](const FVoiceChatChannel& C, 
                                                           const FVoiceChatResult& Res){});
        }
    });
}
```

---

## Module Dependencies

In `YourGame.Build.cs`:

```csharp
PublicDependencyModuleNames.AddRange(new string[]
{
    "Core",
    "CoreUObject",
    "Engine",
    "Voice",                // VoiceChat module
    "VoiceChat",            // IVoiceChat interface
    "OnlineSubsystem",
    "OnlineSubsystemEOS",   // If using EOS backend
    "AudioMixer",           // For audio spatialization
});
```

---

## Known Pitfalls

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| No voice in shipped build | `bEnabled=false` in shipped DefaultEngine.ini | Ensure config is bundled; check pak file |
| Spatial audio not working | Listener position not updated | Call `Set3DPosition()` every frame; check `bPositional=true` |
| Players can hear muted user | Mute not synced across network | Call `MutePlayer()` on all clients or replicate via `PlayerState` |
| High latency echo | Loopback feedback | Disable echo cancellation in DevSettings if VoiceChat-internal AEC is buggy |
| PTT spam / rapid toggle | No cooldown on _Validate | Add timestamp check in `Server_SetTalking_Validate()` |

---

## References

- [Voice Chat Interface (UE5.7 Docs)](https://dev.epicgames.com/documentation/en-us/unreal-engine/voice-chat-interface-in-unreal-engine)
- [Voice Chat with EOS (UE5.7 Docs)](https://dev.epicgames.com/documentation/en-us/unreal-engine/voice-chat-with-epic-online-services)
- [Enhanced Input System (UE5.7 Docs)](https://dev.epicgames.com/documentation/unreal-engine/enhanced-input-in-unreal-engine)
- [Vivox Positional Properties](https://docs.vivox.com/v5/general/unreal/5_18_0/en-us/Unreal/developer-guide/channels/positional-channel-properties.htm)
