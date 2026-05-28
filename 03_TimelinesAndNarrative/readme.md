# 03 - Timelines and Narrative

In block 02 you built the interaction system and wired up the Door so pressing E broadcasts a delegate. The door still does nothing visually. This block fixes that and goes further: you will animate the door using a Timeline, build a reusable narrative trigger system in C++, and hook narrative reactions to player actions.

By the end of this block you should be able to drive smooth procedural animation using Timelines, build a C++ event system that fires narrative triggers based on player actions, and expose the content side of that system cleanly to Blueprint.

---

## Timelines

A `FTimeline` is a runtime object that plays a curve over time and calls a callback every tick while it is running. It is the C++ equivalent of a Blueprint Timeline node.

The canonical use case is the door: rotate from closed to open over 1.5 seconds rather than snapping instantly. A curve drives the rotation, and the Timeline calls your update function every frame with the current curve value.

### What you need

Three things:

1. A `FTimeline` member variable on your Actor.
2. A `UCurveFloat` asset created in the Content Browser that defines the animation shape.
3. A callback function (marked `UFUNCTION`) that receives the current float value each tick.

In `BeginPlay`, bind the curve to the timeline using `AddInterpFloat` and `BindUFunction`. In `Tick`, advance the timeline manually by calling `TickTimeline(DeltaTime)`. When the interaction fires, call `PlayFromStart()`.

The curve asset maps time (X axis) to a normalised value between 0 and 1 (Y axis). Your update callback multiplies that value by the target rotation angle. This separation is deliberate: changing the curve changes the easing, changing the angle property changes how far the door opens, and neither requires touching the other.

### Creating the curve

In the Content Browser: right-click > **Miscellaneous > Curve > CurveFloat**. Add two keys at (0.0, 0.0) and (1.5, 1.0). Set both to **Cubic (Auto)** interpolation for ease-in ease-out. Assign the curve asset to your Door Blueprint's class defaults.

### Why normalised?

The curve always goes from 0 to 1. Your code scales that to the actual output range. One curve can drive a 45-degree door, a 90-degree door, and a 180-degree door — the designer just changes the angle property on the actor, not the curve.

### FTimeline vs UTimelineComponent

`UTimelineComponent` is a component version of the same concept. It is more Blueprint-friendly but requires `CreateDefaultSubobject` and adds a component to the actor. For a single animation on a single actor, `FTimeline` is lighter and self-contained. Use `UTimelineComponent` if you need multiple independent animations on one actor, or if designers need direct Blueprint access to the timeline controls.

### FRotator vs FQuat

The door rotates around one axis (yaw), so `FRotator` works cleanly. For arbitrary-axis rotation, `FQuat` (quaternion) avoids gimbal lock. You do not need to understand quaternion mathematics to use it: treat it as "the correct way to represent a rotation in 3D space."

---

## The Narrative Trigger System

Every escape room reacts to the player. The requirement is a C++ system that fires narrative events when things happen, with content (audio, text, timing) living in Blueprint or data assets rather than hardcoded in C++.

The system has two parts:

**Trigger side:** A component (`UNarrativeTriggerComponent`) that any actor can own and call when something meaningful happens. It broadcasts a named event and optionally notifies a central director.

**Director side:** A single actor (`ANarrativeDirector`) placed in the level that receives event names and routes them to the appropriate response. The routing logic (one-shot filtering, matching event to content) lives in C++. The actual presentation (play audio, show text) is implemented in a Blueprint child.

This separation keeps the trigger sites clean and the reaction logic centralised. A door does not need to know what audio plays when it opens. The director handles that.

### UNarrativeTriggerComponent

Key points to implement:

- Inherits from `UActorComponent`. Tick disabled.
- A `DYNAMIC_MULTICAST_DELEGATE_OneParam` delegate that passes an `FName` event id. Marked `BlueprintAssignable`.
- A `UFUNCTION(BlueprintCallable) void Fire(FName EventId)` that broadcasts the delegate and notifies the director if one exists in the world.
- An `EditAnywhere FName DefaultEventId` for convenience when one actor always fires the same event.

### ANarrativeDirector

Key points to implement:

- A `USTRUCT(BlueprintType)` called `FNarrativeEntry` containing: `FName EventId`, a sound asset pointer, an `FText` for display, and a `bool bOneShot`. All fields need `UPROPERTY` to be editor-visible.
- A `TArray<FNarrativeEntry> NarrativeLines` that designers fill in the editor.
- A private `TSet<FName>` tracking which one-shot events have already fired.
- `UFUNCTION(BlueprintCallable) void ReceiveEvent(FName EventId)` that finds the matching entry, respects the one-shot flag, and calls a `BlueprintImplementableEvent` to hand off to the Blueprint presentation layer.
- A `static ANarrativeDirector* Get(UWorld* World)` that uses `GetAllActorsOfClass` to find the single instance. Cache the result rather than calling it every frame.

### USTRUCT

`FNarrativeEntry` uses `USTRUCT(BlueprintType)`. This is the reflection macro for plain C++ structs. `BlueprintType` makes the struct usable as a variable in Blueprint graphs. Every field you want editable in the Details panel needs its own `UPROPERTY()` inside the struct, exactly like class members.

The `F` prefix on `FNarrativeEntry` is the naming convention for structs covered in block 01.

### Wiring trigger to director

When a meaningful event happens on an actor — a door opens, a key is picked up, a puzzle completes — that actor's `UNarrativeTriggerComponent` calls `Fire` with the relevant event name. `Fire` broadcasts locally and routes to the director globally.

The actor does not know what happens next. It just says something happened. The director decides what to do about it.

---

## Volume-Based Triggers

Not all narrative events come from interaction. Walking into a room should trigger a narrator line. Use an actor with a `UBoxComponent` and bind `OnComponentBeginOverlap` in `BeginPlay`.

Key details:

- Set the box component's collision profile to `"Trigger"` so it generates overlaps without blocking movement.
- In the overlap callback, filter by checking `OtherActor->ActorHasTag("Player")` before routing to the director. Add the tag `"Player"` to the player character's Actor Tags array in the Blueprint class defaults.
- Respect a `bFireOnce` flag with a `bool bHasFired` member.

### Actor Tags

`ActorHasTag` checks a `TArray<FName> Tags` that every `AActor` has, exposed in the Details panel under **Actor > Tags**. It is lightweight filtering without casting.

Use tags for overlap filtering, trace filtering, and loose grouping. Do not use them as a state substitute or as a replacement for proper class hierarchies. `bIsLocked` as a member variable beats `ActorHasTag("Locked")` every time.

---

## Playing Audio from C++

For narrative audio routed through the director, the C++ side calls a `BlueprintImplementableEvent` and passes the audio asset as a parameter. The Blueprint implementation calls the appropriate playback node.

For in-world sounds triggered directly from C++ (a door creak, a lock click), include `Kismet/GameplayStatics.h` and use `UGameplayStatics::PlaySoundAtLocation` for spatialised sound or `UGameplayStatics::PlaySound2D` for narrator voice lines that should not fade with distance.

---

## What to Have Working by End of Block

- `ADoor` animates smoothly using a `FTimeline` driven by a `UCurveFloat`. Opens over 1.5 seconds with ease-in ease-out.
- `UNarrativeTriggerComponent` and `ANarrativeDirector` implemented. At least two named narrative events triggering correctly.
- At least one `ANarrativeVolume` in the level firing a narrative event when the player walks through.
- `ANarrativeDirector` Blueprint child with at least two entries in `NarrativeLines` wired up in Blueprint.
