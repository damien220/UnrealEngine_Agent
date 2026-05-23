# Sky, Atmosphere & Volumetric Fog — Reference

## Overview

UE5's physically-based sky pipeline combines `ASkyAtmosphere`, `USkyLightComponent`, `UDirectionalLightComponent`, optional `UVolumetricCloudComponent`, and `UExponentialHeightFogComponent`. Each component has specific dependencies on the others — misconfiguring them breaks outdoor GI, produces incorrect sky colors, or disables Lumen sky contribution.

---

## Required Component Combination for Physical Sky

All five components are needed for production-quality outdoor lighting:

| Component | Role | Critical Setting |
|---|---|---|
| `ASkyAtmosphere` | Rayleigh + Mie scattering, sky color | Needs `DirectionalLight` tagged as atmosphere sun |
| `UDirectionalLightComponent` | Sun, moon, directional shadow | `bAtmosphereSunLight = true` |
| `USkyLightComponent` | Ambient sky + reflection capture | `bRealTimeCapture = true` (Lumen path) |
| `UVolumetricCloudComponent` | 3D cloud layer with GI | Optional — high GPU cost |
| `UExponentialHeightFogComponent` | Ground-level fog, distance haze | Optional — required for aerial perspective |

### Minimum viable setup (C++)

```cpp
// AMyWeatherActor.h
UCLASS()
class MYGAME_API AMyWeatherActor : public AActor
{
    GENERATED_BODY()
public:
    AMyWeatherActor();

    UPROPERTY(VisibleAnywhere) TObjectPtr<USkyAtmosphereComponent>    SkyAtmosphere;
    UPROPERTY(VisibleAnywhere) TObjectPtr<USkyLightComponent>         SkyLight;
    UPROPERTY(VisibleAnywhere) TObjectPtr<UDirectionalLightComponent> SunLight;
    UPROPERTY(VisibleAnywhere) TObjectPtr<UExponentialHeightFogComponent> HeightFog;
};

// AMyWeatherActor.cpp
AMyWeatherActor::AMyWeatherActor()
{
    SkyAtmosphere = CreateDefaultSubobject<USkyAtmosphereComponent>(TEXT("SkyAtmosphere"));
    RootComponent = SkyAtmosphere;

    SunLight = CreateDefaultSubobject<UDirectionalLightComponent>(TEXT("SunLight"));
    SunLight->SetupAttachment(RootComponent);
    SunLight->bAtmosphereSunLight = true;          // Marks this light as the atmosphere sun
    SunLight->LightSourceAngle = 0.5357f;          // 0.5357° = real sun angular diameter

    SkyLight = CreateDefaultSubobject<USkyLightComponent>(TEXT("SkyLight"));
    SkyLight->SetupAttachment(RootComponent);
    SkyLight->bRealTimeCapture = true;             // Required for Lumen sky contribution
    SkyLight->SourceType = ESkyLightSourceType::SLS_CapturedScene;

    HeightFog = CreateDefaultSubobject<UExponentialHeightFogComponent>(TEXT("HeightFog"));
    HeightFog->SetupAttachment(RootComponent);
}
```

---

## Time-of-Day System

Rotate the `DirectionalLight` pitch to move the sun. `SkyAtmosphere` and `SkyLight` react automatically — no manual sky color updates required.

```cpp
// AMyTimeOfDayManager.h
UCLASS()
class MYGAME_API AMyTimeOfDayManager : public AActor
{
    GENERATED_BODY()
public:
    // 0.0 = midnight, 0.25 = sunrise (6am), 0.5 = noon, 0.75 = sunset (6pm), 1.0 = midnight again
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Time", meta = (ClampMin = "0.0", ClampMax = "1.0"))
    float TimeOfDay = 0.25f;

    // Speed in full day cycles per real second (0.0002 = ~83 min per game day)
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Time")
    float DaySpeed = 0.0002f;

    UPROPERTY(EditAnywhere, Category = "References")
    TObjectPtr<ADirectionalLight> SunActor;

    virtual void Tick(float DeltaTime) override;
    void ApplyTimeOfDay();
};

// AMyTimeOfDayManager.cpp
void AMyTimeOfDayManager::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    TimeOfDay = FMath::Fmod(TimeOfDay + DaySpeed * DeltaTime, 1.0f);
    ApplyTimeOfDay();
}

void AMyTimeOfDayManager::ApplyTimeOfDay()
{
    if (!IsValid(SunActor)) return;

    // Pitch: -90° at midnight (below horizon), +90° at noon
    // TimeOfDay 0.0/1.0 = midnight, 0.5 = noon
    float SunPitch = (TimeOfDay - 0.5f) * 360.f;  // Maps 0–1 to -180 to +180
    SunActor->SetActorRotation(FRotator(SunPitch, 0.f, 0.f));
}
```

**Sky Light recapture**: With `bRealTimeCapture = true`, `USkyLightComponent` automatically recaptures every frame. Without it, call `SkyLight->RecaptureSky()` when the sky changes significantly (e.g., sunrise/sunset transition).

---

## Volumetric Clouds

`UVolumetricCloudComponent` renders a 3D cloud layer using ray-marching. It interacts with Lumen sky lighting but carries significant GPU cost.

```cpp
// Add volumetric clouds
UVolumetricCloudComponent* Clouds = CreateDefaultSubobject<UVolumetricCloudComponent>(TEXT("Clouds"));
Clouds->SetupAttachment(RootComponent);
Clouds->LayerBottomAltitude = 5.0f;    // km above sea level (cloud base)
Clouds->LayerHeight = 10.0f;           // km cloud layer thickness
```

### Cloud material
Assign a `UMaterialInterface` derived from the engine's `M_SimpleVolumetricCloud` as the cloud material. The cloud material controls density, coverage, albedo, and shadow casting.

### Performance profile

```
r.VolumetricCloud 1               -- Enable (default on PC/console, off on mobile)
r.VolumetricCloud.SampleCountMax 16   -- Ray march samples (8 = performance, 16 = quality, 32 = cinematic)
r.VolumetricCloud.ShadowSampleCountMax 8  -- Shadow samples per ray
r.VolumetricCloud 0               -- Disable entirely for a 2–4ms GPU saving
```

**GPU cost reference:**
- `SampleCountMax=8`: ~0.5–1ms GPU (performance target)
- `SampleCountMax=16`: ~1–2ms GPU (quality target)
- `SampleCountMax=32`: ~3–5ms GPU (cinematic — not real-time on mid hardware)

### Lumen interaction

```
When VolumetricCloud shadows are enabled:
  r.VolumetricCloud.EnableAerialPerspectiveSampling 1  -- Aerial perspective from clouds
  DirectionalLight: bCastCloudShadows = true           -- Sun shadows through clouds
  DirectionalLight: CloudShadowExtent = 150000.f       -- Shadow projection distance (UE units)
```

---

## Exponential Height Fog

`UExponentialHeightFogComponent` provides ground-level density fog and aerial perspective.

```cpp
// Set fog parameters in code (or expose as UPROPERTY for designer override)
HeightFog->FogDensity = 0.02f;           // Global density (0.0 = none, 0.1 = thick)
HeightFog->FogHeightFalloff = 0.2f;      // Density dropoff with altitude (higher = thinner at height)
HeightFog->FogInscatteringColor = FLinearColor(0.447f, 0.639f, 0.812f);  // Sky blue tint
HeightFog->FogMaxOpacity = 0.8f;        // Max fog opacity at full density
HeightFog->StartDistance = 0.f;         // Distance from camera where fog begins
HeightFog->FogCutoffDistance = 0.f;     // 0 = no cutoff
```

### Dual-layer fog (UE5.1+)

Add a second fog layer for biome-specific ground mist or colorized night fog:

```cpp
HeightFog->SecondFogData.FogDensity = 0.05f;
HeightFog->SecondFogData.FogHeightFalloff = 1.0f;      // Stays close to ground
HeightFog->SecondFogData.FogHeightOffset = -500.f;     // 5m below base fog height
HeightFog->bEnableVolumetricFog = true;
HeightFog->VolumetricFogDistance = 6000.f;
```

**Aerial perspective**: Enable on `SkyAtmosphere` to make distant mountains blue-shifted realistically:

```
SkyAtmosphere → Aerial Perspective → Start Depth: 0  (in Project Settings or BP detail panel)
r.SkyAtmosphere.AerialPerspectiveLUT.Depth 96   -- Controls aerial perspective distance (km)
```

---

## Calibrating Physical Units

`ASkyAtmosphere` is physically calibrated — use real-world values.

| Parameter | Typical Earth value | Notes |
|---|---|---|
| Rayleigh Scattering Scale | 0.0331 | Controls sky blue |
| Mie Scattering Scale | 0.003996 | Controls haze/smog |
| Mie Anisotropy | 0.8 | Mie forward scatter (0 = isotropic, 1 = forward) |
| Sun Disk Multiplier | 1.0 | Controls sun disk brightness |
| MultiScattering Approximation | 1.0 | Matches single-scatter reference at 1.0 |

For stylized skies (alien worlds, cartoons), these are design parameters — no physical accuracy required.

---

## Lumen Sky Light Integration

For correct Lumen GI from the sky:

1. `USkyLightComponent.SourceType = SLS_CapturedScene` — Lumen reads the captured sky cubemap
2. `bRealTimeCapture = true` — sky cubemap updates every frame (required for time-of-day GI changes)
3. `SkyAtmosphere` feeds sky radiance into Lumen's sky lighting pass automatically
4. `bCastRaytracedShadow = true` on `SkyLight` for soft shadows (hardware RT path only)

**Without `bRealTimeCapture`**: Sky light is static — accurate at capture time but won't respond to time-of-day changes. Use static capture only for performance-constrained builds with baked lighting.

---

## Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `bAtmosphereSunLight = false` on DirectionalLight | Sky atmosphere is flat grey; no scattering | Set `bAtmosphereSunLight = true` on the sun light |
| `bRealTimeCapture = false` with dynamic sky | Sky light doesn't update at sunset | Enable `bRealTimeCapture` or call `RecaptureSky()` on transitions |
| Two DirectionalLights both with `bAtmosphereSunLight = true` | Undefined sky behavior; flicker | Only one DirectionalLight should be the atmosphere sun at a time |
| `VolumetricCloud` enabled with default sample count on mobile | 5–10ms GPU overhead | Set `r.VolumetricCloud 0` on mobile targets |
| SecondFogData without `bEnableVolumetricFog` | Second fog layer visible but no volumetric | Enable volumetric fog on the component |
| No `SkyAtmosphere` present | No physical sky; Directional Light uses fallback color | Always pair `SkyAtmosphere` with `DirectionalLight` in outdoor levels |
