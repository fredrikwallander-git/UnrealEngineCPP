# 04 - Puzzle State Machine

Blocks 01 to 03 gave you the building blocks: C++ classes, the interaction system, the narrative trigger system, and door animation. This block ties them together into the core of the escape room: puzzle logic.

A puzzle in this context is any C++ object that tracks state, validates conditions, and transitions between phases when those conditions are met. The door lock is the simplest form: it has two states (locked, unlocked) and one transition condition (player has the right key). A combination lock is more complex: it has a sequence of steps, validates input at each step, and resets on failure. Both are state machines.

By the end of this block you should be able to build a generic puzzle state machine in C++, connect it to the interaction and narrative systems from blocks 02 and 03, design two interconnected puzzles using the pattern, and explain why a state machine is a better model for puzzle logic than a chain of Blueprint booleans.

Pre-production on the group project should begin now alongside this material. The systems you build here are the systems your project needs.

---

## Why a State Machine

The alternative to a state machine is a collection of booleans and if-chains. This works for one puzzle. It does not work for three interconnected puzzles that each have multiple failure and success conditions, narrative reactions, and dependencies on each other's state.

A state machine makes the following explicit:

- What states exist.
- Which transitions between states are valid.
- What triggers each transition.
- What side effects happen on entry or exit of each state.

Every bit of puzzle complexity that would otherwise become a nested if-chain becomes a named state and a named transition. That is readable, testable, and extensible.

The pattern here is a simplified finite state machine (FSM). You have used FSMs before conceptually, the NavMesh AI in the AI Programming course used Behaviour Trees, which are a more complex variant of the same idea.

---

## A Generic Puzzle Base Class

The project requires that at least two puzzles use the same system, not two separate ad-hoc implementations. The way to enforce this is a `APuzzleBase` C++ class that all puzzles inherit from.

The base class is responsible for:

- Tracking the current state as an `enum`.
- Providing `BlueprintCallable` and `BlueprintImplementableEvent` hooks for state transitions.
- Firing narrative events when states change.

### The state enum

```cpp
UENUM(BlueprintType)
enum class EPuzzleState : uint8
{
    Inactive    UMETA(DisplayName = "Inactive"),
    Active      UMETA(DisplayName = "Active"),
    Solved      UMETA(DisplayName = "Solved"),
    Failed      UMETA(DisplayName = "Failed")
};
```

`UENUM(BlueprintType)` makes the enum visible and usable in Blueprint. `uint8` is required for `UENUM`. `UMETA(DisplayName = "...")` controls the label shown in the editor and Blueprint graph.

Every puzzle starts `Inactive`, becomes `Active` when the player engages with it, reaches `Solved` when the condition is met, and optionally reaches `Failed` if the player makes an invalid move. Not every puzzle needs a `Failed` state but the base class should support it.

### What APuzzleBase owns

Consider what belongs on the base class versus what belongs on the subclass. The base class should own:

- The current `EPuzzleState` and a `SetState` function that fires side effects.
- A `UNarrativeTriggerComponent` for firing narrative events on state change.
- A `bool bResetOnFailure` that subclasses respect.
- A `BlueprintImplementableEvent OnPuzzleStateChanged(EPuzzleState NewState)` so Blueprint children can react visually to any state change.
- A `BlueprintCallable void Activate()` that moves from Inactive to Active.
- A `BlueprintNativeEvent void OnSolved()` with a default implementation that logs and fires a narrative event.
- A reference or dependency system for puzzles that unlock other puzzles (covered below).

The subclass owns the specific input validation logic. A `AKeyPuzzle` knows what keys are required. A `ASequencePuzzle` knows the correct sequence of button presses. `APuzzleBase` knows none of that, it only knows how to transition between states.

---

## Puzzle Interconnection

The project requires at least two interconnected puzzles. "Interconnected" means solving puzzle A changes the state of puzzle B in some way, unlocks it, provides a clue, changes what is required.

The cleanest way to model this in C++ is a dependency list on `APuzzleBase`:

- A `TArray<TObjectPtr<APuzzleBase>> Dependents` exposed as `EditInstanceOnly`. Designers drag the puzzles that this puzzle should unlock into this array in the editor.
- When a puzzle is solved, it iterates `Dependents` and calls `Activate()` on each one.
- This creates a forward-pointing chain: Puzzle A lists Puzzle B as a Dependent. Solving A activates B.

This creates a directed acyclic graph of puzzle dependencies. The designer sets it up in the level editor by wiring actor references. The C++ handles the state propagation. Neither needs to know the specific type of the other puzzle.

The alternative, a `BP_PuzzleManager` Blueprint that manually checks each puzzle's state and enables things, works for two puzzles. It becomes a maintenance problem at three and breaks at five.

---

## A Worked Example: The Key Puzzle

The simplest puzzle that uses the full pattern: the player must find a key pickup and use it on a locked door. The door was built in blocks 02 and 03. This adds the state machine layer.

`AKeyPuzzle` is a subclass of `APuzzleBase`. It represents a lock that requires a specific key. When `Activate()` is called, it enables the door's interactable component and listens for interaction. When the player interacts with the correct key in their inventory, it transitions to `Solved`.

The inventory is covered below. For now, think of it as a `TArray<FName>` on the player character holding collected key ids, which you already built in the block 01 exercises.

The key points:

- `AKeyPuzzle` has a `FName RequiredKeyId` (`EditAnywhere`).
- On `Activate`, it enables the associated door's `UInteractableComponent`.
- On interaction, it checks the player's key array for `RequiredKeyId`.
- If found: call `SetState(EPuzzleState::Solved)`. Remove the key from inventory. Fire a narrative event.
- If not found: fire a "wrong key" narrative event. Do not transition.

None of this is in the door. None of it is in the character. The puzzle object mediates between them.

---

## A Second Puzzle: Sequence Input

A sequence puzzle validates a series of player inputs against a correct sequence. Think of a combination lock, a button sequence on a wall panel, or a set of switches that must be flipped in order.

`ASequencePuzzle` owns:

- A `TArray<FName> CorrectSequence` (`EditDefaultsOnly`). Designers set this in the Blueprint child.
- A `TArray<FName> PlayerInput` that accumulates as the player activates inputs.
- An `int32 CurrentStep` tracking position in the sequence.
- A `UFUNCTION(BlueprintCallable) void SubmitInput(FName InputId)` called by individual button actors when pressed.

`SubmitInput` checks whether `InputId` matches `CorrectSequence[CurrentStep]`. If yes, increment `CurrentStep`. If `CurrentStep == CorrectSequence.Num()`, call `SetState(Solved)`. If no, call `SetState(Failed)` and reset `PlayerInput` and `CurrentStep` if `bResetOnFailure` is true.

The individual buttons are `APuzzleButton` actors with a `UInteractableComponent`. When pressed they call `SequencePuzzle->SubmitInput(MyButtonId)`. The button does not know whether it is correct. It does not know the sequence. It just reports what it is.

This is the same principle as the interaction system: the actor with the interactable component is dumb. The puzzle object has the logic.

---

## A Simple Inventory

The project requires a pickup system. The simplest usable inventory for an escape room is a `TArray<FName>` on the player character. Key ids, item ids, clue tokens, anything the player "has" is just a name in this array.

On `APlayerCharacter`:

```cpp
UPROPERTY(BlueprintReadOnly, Category = "Inventory")
TArray<FName> Inventory;

UFUNCTION(BlueprintCallable, Category = "Inventory")
void AddItem(FName ItemId);

UFUNCTION(BlueprintCallable, Category = "Inventory")
bool HasItem(FName ItemId) const;

UFUNCTION(BlueprintCallable, Category = "Inventory")
bool RemoveItem(FName ItemId);
```

`AKeyPickup` from the exercises calls `AddItem` on the player when collected. The key puzzle calls `HasItem` and `RemoveItem` when validating. The character does not care what the items mean. The puzzle does not care where they came from.

If the project requires showing the inventory in the UI (a HUD display of held items), the `BlueprintReadOnly` property is enough, a UMG widget can read it directly. No additional C++ is needed for a simple display.

---

## State Change Side Effects

When `SetState` transitions to a new state, several things should happen in a consistent order. The pattern:

```
1. Guard: return early if NewState == CurrentState
2. Update CurrentState member
3. Call BlueprintImplementableEvent OnPuzzleStateChanged
4. If Solved: call OnSolved(), then iterate Dependents and call Activate() on each
5. If Failed and bResetOnFailure: reset internal state and return to Active
```

All of this lives in `APuzzleBase::SetState`. Subclasses call `SetState(EPuzzleState::Solved)` and the base class handles the rest. Subclasses never manage dependency activation or Blueprint notification directly, that would duplicate logic across every puzzle type.

The order matters. `OnSolved` fires before dependents activate so the player hears "you solved it" before the next puzzle unlocks. `OnPuzzleStateChanged` fires so Blueprint children can show visual feedback at the correct moment.

---

## The C++ and Blueprint Boundary for Puzzles

This is a good point to be explicit about where the line sits for the escape room project, since the VG criteria require the group to articulate it.

**Lives in C++:**
- Puzzle state and transitions (`EPuzzleState`, `SetState`, transition guards).
- Dependency graph (which puzzles unlock which).
- Input validation logic (correct sequence, required key, combination check).
- Narrative event firing.
- Inventory add, check, remove.

**Lives in Blueprint:**
- Visual feedback when a state changes (animations, material swaps, lights changing colour).
- Audio responses (sound effects on success, failure, unlock).
- Specific puzzle configurations (the correct sequence, the required key id, the narrative event names).
- The actual assets (meshes, sounds, particle effects).

The test is: could a designer change the puzzle's correct answer without a programmer? If yes, the answer lives in Blueprint (as a `EditDefaultsOnly` property on a Blueprint child). Could a programmer break the state machine by editing a Blueprint? If yes, something is in the wrong place.

---

## What to Have Working by End of Block

- `APuzzleBase` implemented with `EPuzzleState`, `SetState`, `BlueprintImplementableEvent OnPuzzleStateChanged`, `UNarrativeTriggerComponent`, and a basic dependency list.
- `AKeyPuzzle` implemented as a subclass. Requires the key pickup from the exercises.
- `ASequencePuzzle` implemented with at least three-step validation. A matching set of `APuzzleButton` actors.
- Both puzzles placed in the level. Solving puzzle 1 (the key) activates puzzle 2 (the sequence).
- `APlayerCharacter` has `AddItem`, `HasItem`, `RemoveItem`. The key pickup uses `AddItem`.
- At minimum one `BlueprintImplementableEvent` on a puzzle shows visual feedback in a Blueprint child.