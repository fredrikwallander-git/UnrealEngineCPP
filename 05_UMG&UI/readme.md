# 05 - UMG and UI in C++

Blocks 01 to 04 covered the gameplay layer: actors, interaction, narrative, and puzzle state machines. The player can now engage with a world that reacts to them. What is missing is the interface layer, the feedback the player receives on screen that communicates game state clearly: what they can interact with, what they are carrying, and whether they have won.

This block covers how to drive UMG widgets from C++, how to build an interact prompt that appears when the player looks at something interactable, how to display the player's inventory, and how to implement a win state screen. These are not optional polish,  the grading criteria require a win state, and a game without an interact prompt is nearly unplayable in a first-person or third-person context.

By the end of this block you should be able to create and manage UMG widgets from C++, drive widget content from C++ data, implement a functional interact prompt tied to the existing interaction system, build a simple HUD showing inventory state, and trigger a win screen from a puzzle event.

---

## Adding UMG to the Build

Open `EscapeRoomLab.Build.cs` and add `"UMG"` and `"Slate"` to the dependency list if not present:

```csharp
PublicDependencyModuleNames.AddRange(new string[]
{
    "Core", "CoreUObject", "Engine", "InputCore", "EnhancedInput", "UMG", "Slate"
});
```

Build after adding this. Any class that uses UMG types must be in a module that lists these dependencies.

---

## How UMG Works from C++

A UMG widget has two parts that mirror each other: a `UUserWidget` C++ class that holds logic and data, and a Blueprint child (a Widget Blueprint) that holds the visual layout built in the UMG designer.

The C++ class is responsible for:
- Declaring the properties and functions the widget needs.
- Binding those to the visual elements via `UPROPERTY(meta = (BindWidget))`.
- Providing data from the game's state.

The Widget Blueprint is responsible for:
- The visual layout, which panels, text blocks, images, and buttons exist.
- The animations, styles, and colours.
- Nothing related to game logic.

The relationship is the same as with Actor and Blueprint child: C++ defines behaviour, Blueprint defines appearance.

### BindWidget

`meta = (BindWidget)` on a `UPROPERTY` tells the UMG system that a C++ pointer corresponds to a named widget in the Blueprint layout. The widget in the Blueprint must have exactly the same name as the C++ variable.

```cpp
UPROPERTY(meta = (BindWidget))
TObjectPtr<UTextBlock> InteractPromptText;
```

If a Widget Blueprint has a `UTextBlock` named `InteractPromptText`, UMG automatically wires the pointer. If the names do not match, the Blueprint will show a binding error and the pointer will be null at runtime.

`BindWidgetOptional` is available for widgets that may or may not exist in the layout, the pointer is null if absent rather than causing an error. Use it for elements that some Widget Blueprint children have and others do not.

### Creating and Adding Widgets

Widgets are created at runtime using `CreateWidget` and added to the viewport using `AddToViewport`. Both are typically called from the PlayerController or the Character.

```cpp
#include "Blueprint/UserWidget.h"

// In the Character or PlayerController:
UPROPERTY(EditDefaultsOnly, Category = "UI")
TSubclassOf<UUserWidget> HUDWidgetClass;

TObjectPtr<UUserWidget> HUDWidget;
```

In `BeginPlay`:

```cpp
if (HUDWidgetClass)
{
    HUDWidget = CreateWidget<UUserWidget>(GetWorld(), HUDWidgetClass);
    if (HUDWidget)
        HUDWidget->AddToViewport();
}
```

`CreateWidget` takes either a `UWorld*`, a `APlayerController*`, or a `UGameInstance*` as the owning object. Use `APlayerController*` when the widget is player-specific. Use `UWorld*` for world-scoped widgets. The owning object controls the widget's lifetime, when the owner is destroyed, the widget is also cleaned up.

`AddToViewport` takes an optional Z-order integer. Higher Z-order renders on top. Use this when layering multiple widgets (HUD at 0, interact prompt at 1, win screen at 10).

### Removing Widgets

```cpp
HUDWidget->RemoveFromParent();
HUDWidget = nullptr;
```

`RemoveFromParent` removes the widget from the viewport without destroying it. Setting the pointer to null drops the reference and allows garbage collection.

---

## The Interact Prompt

The interact prompt appears when the player is looking at something with an enabled `UInteractableComponent` and disappears otherwise. It shows the `InteractPrompt` text from the component.

The architecture: the Character already tracks `FocusedInteractable` every tick. The HUD widget reads from it.

### C++ Widget Class

```cpp
// EscapeRoomHUD.h
#pragma once

#include "CoreMinimal.h"
#include "Blueprint/UserWidget.h"
#include "EscapeRoomHUD.generated.h"

class UTextBlock;
class UWidget;

UCLASS()
class ESCAPEROOMLAB_API UEscapeRoomHUD : public UUserWidget
{
    GENERATED_BODY()

public:
    // Called every frame from the Character to push current state.
    void UpdateInteractPrompt(const FText& Prompt, bool bVisible);
    void UpdateInventoryCount(int32 Count);

protected:
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UTextBlock> InteractPromptText;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UWidget> InteractPromptContainer;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UTextBlock> InventoryCountText;
};
```

```cpp
// EscapeRoomHUD.cpp
#include "EscapeRoomHUD.h"
#include "Components/TextBlock.h"
#include "Components/Widget.h"

void UEscapeRoomHUD::UpdateInteractPrompt(const FText& Prompt, bool bVisible)
{
    if (InteractPromptText)
        InteractPromptText->SetText(Prompt);

    if (InteractPromptContainer)
        InteractPromptContainer->SetVisibility(
            bVisible ? ESlateVisibility::Visible : ESlateVisibility::Hidden);
}

void UEscapeRoomHUD::UpdateInventoryCount(int32 Count)
{
    if (InventoryCountText)
        InventoryCountText->SetText(
            FText::FromString(FString::Printf(TEXT("Items: %d"), Count)));
}
```

### Updating from Tick

In the Character's `Tick`, after the line trace:

```cpp
if (HUDWidget)
{
    UEscapeRoomHUD* HUD = Cast<UEscapeRoomHUD>(HUDWidget);
    if (HUD)
    {
        bool bHasFocus = FocusedInteractable && FocusedInteractable->bIsEnabled;
        FText Prompt = bHasFocus
            ? FocusedInteractable->InteractPrompt
            : FText::GetEmpty();
        HUD->UpdateInteractPrompt(Prompt, bHasFocus);
        HUD->UpdateInventoryCount(Inventory.Num());
    }
}
```

### Widget Blueprint Layout

Create a Widget Blueprint child of `UEscapeRoomHUD`. Name it `WBP_HUD`.

In the UMG designer:
- Add a **Canvas Panel** as root.
- Add a **Vertical Box** or **Border** named `InteractPromptContainer` anchored at the bottom centre. This is the container that shows and hides.
- Inside it, add a **Text Block** named `InteractPromptText`. Set a default text like "Press E".
- Add a **Text Block** named `InventoryCountText` anchored at the top left.

The names in the Blueprint must exactly match the `UPROPERTY` names in the C++ class.

In the Character Blueprint class defaults, assign `WBP_HUD` to the `HUDWidgetClass` property.

---

## NativeConstruct and NativeTick

`UUserWidget` has two C++ lifecycle functions worth knowing:

**`NativeConstruct`**, called when the widget is created and added to the viewport. Equivalent to `BeginPlay` for widgets. Use it for one-time setup: setting initial text, binding delegates, caching references.

**`NativeTick`**, called every frame if the widget has ticking enabled. Avoid using it if possible. Polling game state from a widget every tick is a pattern that scales poorly. Push data from the game to the widget (call `UpdateInteractPrompt` from Character Tick) rather than having the widget pull it.

```cpp
protected:
    virtual void NativeConstruct() override;
```

---

## The Win Screen

The win screen appears when the final puzzle is solved. It is a separate widget from the HUD, shown on top of everything else.

### C++ Class

```cpp
UCLASS()
class ESCAPEROOMLAB_API UWinScreen : public UUserWidget
{
    GENERATED_BODY()

public:
    void Show(float DisplayTime = 0.0f);

protected:
    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UTextBlock> WinMessageText;

    UPROPERTY(meta = (BindWidget))
    TObjectPtr<UButton> QuitButton;

    virtual void NativeConstruct() override;

    UFUNCTION()
    void OnQuitClicked();
};
```

```cpp
void UWinScreen::NativeConstruct()
{
    Super::NativeConstruct();
    if (QuitButton)
        QuitButton->OnClicked.AddDynamic(this, &UWinScreen::OnQuitClicked);
}

void UWinScreen::Show(float DisplayTime)
{
    AddToViewport(10);
    SetVisibility(ESlateVisibility::Visible);

    if (WinMessageText)
        WinMessageText->SetText(FText::FromString(TEXT("You escaped.")));
}

void UWinScreen::OnQuitClicked()
{
    UKismetSystemLibrary::QuitGame(this, nullptr, EQuitPreference::Quit, false);
}
```

### Showing the Win Screen from the GameMode

The final puzzle calls `SetState(Solved)` which fires the `"PuzzleSolved"` narrative event. The `ANarrativeDirector` Blueprint implementation can call a function on the GameMode to show the win screen:

```cpp
// EscapeRoomGameMode.h
UFUNCTION(BlueprintCallable, Category = "Game")
void TriggerWin();

// EscapeRoomGameMode.cpp
void AEscapeRoomGameMode::TriggerWin()
{
    APlayerController* PC = GetWorld()->GetFirstPlayerController();
    if (!PC) return;

    // Show the win screen widget.
    if (WinScreenClass)
    {
        UWinScreen* WinScreen = CreateWidget<UWinScreen>(PC, WinScreenClass);
        if (WinScreen)
            WinScreen->Show();
    }

    // Optional: disable player input.
    PC->SetInputMode(FInputModeUIOnly());
    PC->bShowMouseCursor = true;
}
```

Add `TSubclassOf<UWinScreen> WinScreenClass` as an `EditDefaultsOnly` property on the GameMode. Assign the Widget Blueprint in the GameMode Blueprint child's class defaults.

In the `ANarrativeDirector` Blueprint's `OnNarrativeLine` implementation, check the event id: if it is `"EscapeComplete"`, call `GetGameMode > Cast to EscapeRoomGameMode > TriggerWin`.

---

## Input Mode

When showing a menu or win screen, the player should not be able to keep moving. Unreal has three input modes:

```cpp
// Game only, no cursor, all input goes to the game
PC->SetInputMode(FInputModeGameOnly());
PC->bShowMouseCursor = false;

// UI only, cursor visible, input goes to widgets
PC->SetInputMode(FInputModeUIOnly());
PC->bShowMouseCursor = true;

// Game and UI, cursor visible, both game and widgets receive input
PC->SetInputMode(FInputModeGameAndUI());
PC->bShowMouseCursor = true;
```

Switch to `FInputModeUIOnly` when showing the win screen so the player can click the quit button. Switch back to `FInputModeGameOnly` if the game has a pause menu that can be dismissed.

---

## Common UMG C++ Mistakes

**Forgetting to include the module.** If `UTextBlock` is not found, `"UMG"` is missing from `Build.cs`.

**Name mismatch with BindWidget.** The C++ variable name must exactly match the widget name in the Blueprint designer. Case-sensitive. A mismatch shows as a warning on the Widget Blueprint and the pointer is null at runtime.

**Calling SetText with a raw string.** `UTextBlock::SetText` takes `FText`, not `FString`. Use `FText::FromString` or `FText::Format`.

**Creating widgets in the constructor.** `CreateWidget` requires a valid world context. Call it in `BeginPlay` or later, never in the constructor.

**Not null-checking BindWidget pointers.** Even with `BindWidget`, the pointer can be null if the Widget Blueprint layout does not have the named element. Always null-check before calling methods on it.

---

## What to Have Working by End of Block

- `UEscapeRoomHUD` implemented with interact prompt and inventory count. Prompt appears when looking at an interactable, disappears otherwise. Inventory count updates when items are picked up.
- `UWinScreen` implemented with a win message and a quit button.
- Final puzzle in the level fires `"EscapeComplete"` narrative event. Director Blueprint calls `TriggerWin` on the GameMode. Win screen appears.
- Mouse cursor visible and player input locked to UI on win.
- Widget Blueprint layouts created for both widgets with correct element names.
- Everything pushed to your GitHub repository.
