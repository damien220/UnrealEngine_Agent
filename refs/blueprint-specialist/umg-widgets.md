# UMG Widget System: UUserWidget Lifecycle, Pooling, and Performance

## UUserWidget Lifecycle Events

The `UUserWidget` class in Unreal Engine 5 provides a well-defined initialization and destruction sequence. Understanding the exact timing of each event is critical for correct binding setup, widget tree construction, and resource cleanup.

### Lifecycle Order

Three native virtual functions define the complete lifecycle:

1. **`NativeOnInitialized()`** — Called exactly once at runtime on the first frame, *before* the widget is added to the viewport. The widget object exists but has not been rendered. This is the correct place to:
   - Bind to delegates on properties marked with `BindWidget` (these are guaranteed to exist)
   - Set up one-time event listeners on owned UObjects
   - Initialize references that must survive widget removal/re-addition

   **Important**: Do NOT cache references to named child widgets here — the widget tree may not be fully constructed until `NativeConstruct()`. Use `BindWidget` properties instead.

2. **`NativeConstruct()`** — Called every time the widget is added to the viewport (multiple times if the widget is removed and re-added). This is where:
   - Child widget references are safe to access (tree is built)
   - Bindings to child widget events can be created
   - Event-driven updates should be hooked up
   - Visual initialization occurs
   
   Blueprint equivalent: `Construct` event (fired on every addition to viewport).

3. **`NativeDestruct()`** — Called every time the widget is removed from the viewport. Counterpart to `NativeConstruct()`. Used for:
   - Cleaning up active event listeners bound in `NativeConstruct()`
   - Stopping `Tick` if enabled
   - Releasing child widget references if pooling manually

### Key Distinction: Initialize vs. NativeOnInitialized

The `Initialize()` function is called internally by the engine and handles UMG initialization. Do NOT override it directly — use `NativeOnInitialized()` instead. The public `Initialize()` method invokes the entire setup chain, including `NativeOnInitialized()`.

---

## Widget Update Patterns

### Property Binding (Expensive)

**Pattern**: Direct attribute binding on a widget property in the designer.

Property bindings are evaluated **every frame** by polling the bound function or member variable. For example, binding `Health` directly to a text widget's text attribute will evaluate the binding every frame, marking the widget as **Volatile** and forcing a re-paint even if the value hasn't changed.

**Cost**: High CPU usage, especially with complex binding expressions.

**Preferred Use**: Only for very cheap reads (e.g., displaying a single integer score that changes rarely).

### Event-Driven Updates (Preferred)

**Pattern**: Override `NativeConstruct()` and bind to delegates.

```cpp
void UMyHealthWidget::NativeConstruct()
{
    Super::NativeConstruct();
    
    if (ACharacter* OwnerChar = Cast<ACharacter>(GetOwningPlayerPawn()))
    {
        // Assuming character has a health delegate
        if (IHealthInterface* HealthInt = Cast<IHealthInterface>(OwnerChar))
        {
            HealthInt->OnHealthChanged.AddDynamic(this, &UMyHealthWidget::OnHealthUpdated);
        }
    }
}

void UMyHealthWidget::OnHealthUpdated(float NewHealth)
{
    HealthText->SetText(FText::AsNumber(NewHealth));
}

void UMyHealthWidget::NativeDestruct()
{
    if (ACharacter* OwnerChar = Cast<ACharacter>(GetOwningPlayerPawn()))
    {
        if (IHealthInterface* HealthInt = Cast<IHealthInterface>(OwnerChar))
        {
            HealthInt->OnHealthChanged.RemoveDynamic(this, &UMyHealthWidget::OnHealthUpdated);
        }
    }
    Super::NativeDestruct();
}
```

This pattern only updates the widget when data changes, not every frame.

### Tick (Avoid in UUserWidget)

**Anti-pattern**: Enabling `bCanEverTick` in the constructor and using `NativeTick()`.

UUserWidget ticking is expensive and should be avoided. If you need per-frame updates, override `NativeUpdateAnimation()` in an animation blueprint instead. If `NativeTick` is absolutely unavoidable (e.g., animating progress over time):

```cpp
FMyHealthWidget::FMyHealthWidget(const FObjectInitializer& Initializer)
    : Super(Initializer)
{
    bCanEverTick = false;  // Default — keep this way
}
```

---

## SynchronizeProperties Override

`SynchronizeProperties()` is called:
- In the editor when a property is changed in the designer
- In the editor when the Blueprint is compiled
- At runtime during widget construction (before `NativeConstruct`)

Override this function to apply property changes to child widgets for design-time preview:

```cpp
void UMyCustomWidget::SynchronizeProperties()
{
    Super::SynchronizeProperties();
    
    if (TitleText)
    {
        TitleText->SetText(TitleTextValue);
    }
    
    if (ContainerPanel)
    {
        ContainerPanel->SetColorAndOpacity(BackgroundColor);
    }
}
```

This ensures the widget's appearance in the editor matches runtime behavior as properties are changed in the details panel.

---

## Widget Pooling

### FUserWidgetPool for Manual Pooling

For widgets created and destroyed frequently (e.g., inventory items, ability buttons), use `FUserWidgetPool`:

```cpp
// In header
FUserWidgetPool InventoryItemPool;

// In BeginPlay / initialization
void AMyHUD::InitializePool()
{
    InventoryItemPool.ReleaseAllSlots();
    InventoryItemPool.Reserve(50, TSubclassOf<UInventoryItemWidget>());
}

// When creating a widget
UInventoryItemWidget* ItemWidget = Cast<UInventoryItemWidget>(
    InventoryItemPool.GetOrCreateInstance(TSubclassOf<UInventoryItemWidget>(UInventoryItemWidget::StaticClass()))
);
if (ItemWidget)
{
    ItemWidget->SetItem(Item);
    MyContainer->AddChild(ItemWidget);
}

// When destroying a widget (returning to pool)
void AMyHUD::ReturnItemToPool(UInventoryItemWidget* ItemWidget)
{
    ItemWidget->RemoveFromParent();
    InventoryItemPool.Release(ItemWidget);
}

// Cleanup in destructor
void AMyHUD::BeginDestroy()
{
    InventoryItemPool.ReleaseAllSlots();
    Super::BeginDestroy();
}
```

**Important**: Call `ReleaseAllSlots()` in your widget's `ReleaseSlateResources()` to prevent circular reference leaks.

### Automatic Pooling with List Views

`UListView`, `UTileView`, and `UTreeView` automatically pool entry widgets:

```cpp
// In your list view data source
void AMyCharacterScreen::RefreshCharacterList()
{
    TArray<UCharacterData*> Characters = GetCharacterData();
    MyListView->ClearListItems();
    
    for (UCharacterData* Char : Characters)
    {
        MyListView->AddItem(Char);  // ListView handles pooling internally
    }
}
```

These virtualized list widgets:
- Only create entry widgets for visible items (e.g., 5 on-screen from 200 total)
- Automatically recycle widgets as the list scrolls
- Maintain performance even with thousands of items

Do NOT manually pool list view entries — the widget handles it.

---

## Invalidation Panels and Caching

### UInvalidationBox for Static Regions

`UInvalidationBox` caches the render commands of child widgets and only redraws them when explicitly invalidated. Use this for UI regions that don't change every frame:

```cpp
// In designer: place static UI (title, icons) inside an InvBox
// The box caches the draw commands of children and uses the cache
// unless children change
```

**Key benefit**: Expensive geometry/shader operations (complex layouts, text rendering) are cached instead of redrawn every frame.

**When to use**:
- Static scoreboard overlays
- Menu panels that change infrequently
- Loading screens with static text and images

**When NOT to use**:
- Widgets with per-frame changes (health bars, timers, animated progress)
- Widgets with `SetRenderTransform` applied (bypasses invalidation)

### Manual Invalidation

If a child widget's appearance changes and it's inside an `UInvalidationBox`, manually invalidate:

```cpp
void UMyScoreWidget::UpdateScore(int32 NewScore)
{
    ScoreText->SetText(FText::AsNumber(NewScore));
    
    if (UInvalidationBox* InvBox = Cast<UInvalidationBox>(GetParent()))
    {
        InvBox->Invalidate(EInvalidateWidgetReason::Layout);
    }
}
```

**Note**: Children with `SetRenderTransform` (transforms, opacity) bypass `UInvalidationBox` caching entirely. Use invalidation explicitly for these cases.

---

## Common UMG API Reference

### Widget Creation (Correct Pattern)

```cpp
// In APlayerController or from Pawn with controller
APlayerController* PC = GetWorld()->GetFirstPlayerController();
UMyMenuWidget* Menu = CreateWidget<UMyMenuWidget>(PC, UMyMenuWidget::StaticClass());
if (Menu)
{
    Menu->AddToViewport(10);  // ZOrder = 10
}

// Alternative from character
if (APlayerController* PC = GetController<APlayerController>())
{
    UMyHUDWidget* HUD = CreateWidget<UMyHUDWidget>(PC, UMyHUDWidget::StaticClass());
    HUD->AddToViewport(0);
}
```

**Important**: Never use `NewObject<UUserWidget>()` or `SpawnActor()` for widgets. `APlayerController::CreateWidget<T>()` is the only correct factory.

### Common Functions

- **`APlayerController::CreateWidget<T>()`** — Factory function (correct, only way)
- **`UUserWidget::AddToViewport(int32 ZOrder = 0)`** — Add to screen
- **`UUserWidget::RemoveFromParent()`** — Remove from screen
- **`UUserWidget::GetOwningPlayer()`** — Get the APlayerController that owns this widget
- **`UUserWidget::GetOwningPlayerPawn()`** — Get the pawn of the owning player
- **`UWidgetLayoutLibrary::GetViewportSize()`** — Get screen dimensions
- **`UWidgetBlueprintLibrary::SetInputMode_*()`** — Set input mode (UI-only, game, etc.)

### Helper Libraries

Include in `.Build.cs`:
```cpp
PublicDependencyModuleNames.AddRange(new string[] { "UMG", "Slate", "SlateCore" });
```

Headers:
```cpp
#include "Blueprint/UserWidget.h"
#include "Blueprint/WidgetBlueprintLibrary.h"
#include "Blueprint/WidgetLayoutLibrary.h"
#include "Components/Image.h"
#include "Components/TextBlock.h"
#include "Components/Button.h"
#include "Components/ListView.h"
```

---

## Performance Guidelines

| Rule | Cost | Solution |
|------|------|----------|
| Property binding on visual attribute | High per-frame | Use event-driven updates instead |
| UUserWidget with `bCanEverTick = true` | Medium per-frame | Use `NativeConstruct()` event binding only |
| Unbound event listeners in `NativeConstruct()` | Memory leak | Always unbind in `NativeDestruct()` |
| Complex child layouts in every frame | High | Wrap in `UInvalidationBox` |
| List view with > 100 items, no pooling | High | Use `UListView` (auto-pools) |
| `NewObject<UUserWidget>()` instead of `CreateWidget()` | Crash risk | Always use `APlayerController::CreateWidget<T>()` |

---

## Summary

- **Lifecycle**: `NativeOnInitialized()` (once) → `NativeConstruct()` (per viewport add) → `NativeDestruct()` (per removal)
- **Updates**: Event-driven via delegate binding in `NativeConstruct()`; avoid property binding and Tick
- **Pooling**: Use `FUserWidgetPool` for manual pooling; `UListView/UTileView/UTreeView` auto-pool
- **Caching**: `UInvalidationBox` for static regions; manually invalidate on changes
- **Factory**: Always `APlayerController::CreateWidget<T>()`, never `NewObject()`

