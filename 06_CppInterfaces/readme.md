# 06 - C++ Interfaces

Blocks 02 through 05 built the interaction system using a component. Any actor that has a `UInteractableComponent` can be interacted with. The player detects the component using `FindComponentByClass` and calls `Interact` on it. This works and it is a valid pattern.

There is a second approach: C++ interfaces. An interface defines a contract. Any class that implements the interface promises to provide certain functions. The player can interact with any actor that implements the interface, without caring what type the actor is or whether it has a specific component.

Understanding both patterns and being able to articulate the trade-offs between them is a VG requirement. This block covers C++ interfaces in Unreal, when to prefer them over the component pattern, and how to use them in the escape room context.

---

## What is an Interface?

An interface is a pure contract. It declares functions but does not implement them. Any class that implements the interface must provide implementations for those functions. Code that holds a reference to the interface can call those functions without knowing the concrete type.

In standard C++ an interface is typically an abstract class with only pure virtual functions. In Unreal, interfaces have two classes: a `UInterface` class (which Unreal needs for reflection) and an `IInterface` class (which your gameplay code actually uses).

```cpp
// IInteractable.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/Interface.h"
#include "IInteractable.generated.h"

// MinimalAPI means only the minimum required symbols from this class are exported
// from the module (essentially just the type information the linker needs).

// Since UInteractable is boilerplate that gameplay code never calls directly,
// exporting its full interface would be wasteful. MinimalAPI is standard practice
// for all UInterface boilerplate classes.

// Blueprintable allows Blueprint classes to implement this interface via Class Settings.
UINTERFACE(MinimalAPI, Blueprintable)
class UInteractable : public UInterface
{
    GENERATED_BODY()
};

class ESCAPEROOMLAB_API IInteractable
{
    GENERATED_BODY()

public:
    // These are the dispatch functions. Call these from outside the class.
    // The UHT generates their implementation -- do not define them yourself.
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category = "Interaction")
    void Interact(AActor* Interactor);

    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category = "Interaction")
    FText GetInteractPrompt() const;

    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category = "Interaction")
    bool CanInteract() const;

    // These are the implementation functions. Override these in your class.
    // The _Implementation suffix is the Unreal convention for BlueprintNativeEvent bodies.
   
    virtual void Interact_Implementation(AActor* Interactor) {}
    virtual FText GetInteractPrompt_Implementation() const { return FText::GetEmpty(); }
    virtual bool CanInteract_Implementation() const { return true; }
};
```

The `UInteractable` class is boilerplate that Unreal requires. You never instantiate it or inherit from it in gameplay code. The `IInteractable` class is what you actually work with.

`Blueprintable` on the `UINTERFACE` macro makes the interface implementable in Blueprint as well as C++.

`BlueprintNativeEvent` on each function means there is a C++ default implementation (the `_Implementation` suffix) that Blueprint can optionally override. This is the standard choice for interface functions that need both C++ defaults and Blueprint flexibility.

---

## Implementing an Interface in C++

A class implements an interface by inheriting from the `I` class alongside its normal inheritance:

```cpp
UCLASS()
class ESCAPEROOMLAB_API ADoor : public AActor, public IInteractable
{
    GENERATED_BODY()

public:
    virtual void Interact_Implementation(AActor* Interactor) override;
    virtual FText GetInteractPrompt_Implementation() const override;
    virtual bool CanInteract_Implementation() const override;
};
```

The functions are declared with the `_Implementation` suffix, covered in the "_What is an Interface section above_". 

Override these in your implementing class. In the cpp:

```cpp
void ADoor::Interact_Implementation(AActor* Interactor)
{
    // Door-specific interaction logic.
}

FText ADoor::GetInteractPrompt_Implementation() const
{
    return FText::FromString(bIsOpen ? TEXT("Close Door") : TEXT("Open Door"));
}

bool ADoor::CanInteract_Implementation() const
{
    return !bIsAnimating;
}
```

---

## Calling Interface Functions

You can call interface functions two ways. The first is through a cast:

```cpp
if (IInteractable* Interactable = Cast<IInteractable>(HitActor))
{
    if (Interactable->Execute_CanInteract(HitActor))
        Interactable->Execute_Interact(HitActor, this);
}
```

The second is through `Execute_` static functions that the UHT generates. These are the correct way to call interface functions when the implementing class might be a Blueprint rather than a C++ class:

```cpp
if (HitActor->Implements<UInteractable>())
{
    if (IInteractable::Execute_CanInteract(HitActor))
        IInteractable::Execute_Interact(HitActor, this);
}
```

`Execute_` functions go through the Blueprint dispatch system, so they work correctly whether the interface is implemented in C++ or Blueprint. Always use `Execute_` when the implementing class might be a Blueprint.

`Implements<UInteractable>()` checks whether an actor implements the interface. Use this instead of `Cast` when you only need to check, not to call methods.

---

## Updating the Line Trace

With the interface approach, the character's line trace does not look for a component. It checks whether the hit actor implements the interface:

```cpp
void AEscaperoomLabCharacter::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    FocusedActor = nullptr;

    FVector Start = GetActorLocation() + FVector(0, 0, BaseEyeHeight);
    FVector End   = Start + GetActorForwardVector() * InteractRange;

    FHitResult Hit;
    FCollisionQueryParams Params;
    Params.AddIgnoredActor(this);

    if (GetWorld()->LineTraceSingleByChannel(Hit, Start, End, ECC_Visibility, Params))
    {
        AActor* HitActor = Hit.GetActor();
        if (HitActor && HitActor->Implements<UInteractable>())
        {
            FocusedActor = HitActor;
        }
    }
}

void AEscaperoomLabCharacter::Interact()
{
    if (!FocusedActor) return;
    if (IInteractable::Execute_CanInteract(FocusedActor))
        IInteractable::Execute_Interact(FocusedActor, this);
}
```

`FocusedActor` replaces `FocusedInteractable`. The character stores a reference to the actor, not a reference to a component.

The HUD update changes accordingly:

```cpp
if (FocusedActor && FocusedActor->Implements<UInteractable>())
{
    FText Prompt = IInteractable::Execute_GetInteractPrompt(FocusedActor);
    HUDWidget->UpdateInteractPrompt(Prompt, true);
}
else
{
    HUDWidget->UpdateInteractPrompt(FText::GetEmpty(), false);
}
```

---

## Interface vs Component: The Trade-off

Both patterns solve the same problem. The choice depends on your priorities.

**Component pattern (`UInteractableComponent`):**

A component is data and behaviour attached to an actor. It is the right choice when:
- Multiple different actor types need identical shared behaviour and you want that behaviour in one place.
- The behaviour needs to be toggled at runtime (enable and disable the component).
- Designers need to configure the behaviour per-instance in the editor (the component has `EditAnywhere` properties).
- Blueprint-only actors (no C++ class) need the behaviour, you can add a component in the Blueprint editor.

Drawbacks:
- `FindComponentByClass` is a reflection query. It works but has a cost if called every frame on many actors.
- The actor and the component are separate objects. There is an extra indirection.

**Interface pattern (`IInteractable`):**

An interface is a contract enforced at compile time. It is the right choice when:
- You want the compiler to enforce that implementing classes provide specific functions.
- The implementation varies significantly per type (a door does different things than a puzzle button or a pickup).
- You want Blueprint classes to be able to implement the interface without a C++ base class.
- You prefer the interaction logic to live directly on the actor rather than on a separate component.

Drawbacks:
- A Blueprint-only actor cannot implement a C++ interface unless it has a C++ parent class (or you use the Blueprint interface system separately).
- There is no runtime enable/disable equivalent. `CanInteract` must be checked explicitly.
- Adding new interface functions requires updating every implementing class.

For the escape room project either pattern is valid. The component approach from blocks 02 to 04 is simpler to set up initially and easier to add to Blueprint-only actors. The interface approach is cleaner in C++ and scales better when implementing types diverge significantly.

---

## Implementing in Blueprint

Because the interface uses `Blueprintable` and `BlueprintNativeEvent`, a Blueprint actor can implement it without a C++ parent class:

1. Open a Blueprint actor.
2. In the Class Settings panel, under **Interfaces > Implemented Interfaces**, click **Add** and search for `Interactable`.
3. Functions with the interface name appear in the **My Blueprint** panel under Interfaces.
4. Double-click `Interact` to implement it. The graph opens with an `Event Interact` node.

The `Execute_Interact` call from the character works correctly on this Blueprint actor because it routes through the Blueprint virtual dispatch system.

---

## A Note on Multiple Interfaces

A class can implement multiple interfaces:

```cpp
class ESCAPEROOMLAB_API APuzzleButton
    : public AActor
    , public IInteractable
    , public IHighlightable
{
    GENERATED_BODY()
    // ...
};
```

This is one of the main advantages of interfaces over base classes. A class cannot inherit from two `AActor` subclasses, but it can implement any number of interfaces.

---

## What to Have Working by End of Block

Choose one of the following and implement it fully. Be prepared to explain your choice in the technical document.

**Option A:** Keep the component pattern from blocks 02 to 04.

**Option B:** Migrate the interaction system to use `IInteractable`. Update the Door, KeyPickup, and at least one puzzle button to implement the interface. Update the character to use `Implements<UInteractable>` and `Execute_` calls instead of `FindComponentByClass`.

Both options result in a working game.