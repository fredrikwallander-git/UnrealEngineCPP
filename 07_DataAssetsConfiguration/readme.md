# 07 - Data Assets and Configuration

Blocks 01 through 06 covered architecture: how to structure actors, components, interfaces, state machines, and UI. One thing has been implicit throughout: configuration values live either hardcoded in C++ or scattered across Blueprint class defaults. For a small project this is manageable. For a larger one it becomes a maintenance problem.

This block introduces two Unreal systems for separating data from code: `UDataAsset` and `UDataTable`. Both let designers configure game content without touching C++ or even Blueprint graphs. The goal is a clear boundary where programmers define the shape of data and designers fill it in.

By the end of this block you should be able to create `UDataAsset` subclasses, expose them to the editor, read from them at runtime, and use `UDataTable` with a row struct for tabular configuration like puzzle definitions or item databases.

---

## The Problem With Inline Configuration

Consider the sequence puzzle from block 04. The correct sequence is set as an `EditDefaultsOnly TArray<FName>` on the Blueprint child's class defaults. This works, but:

- To see all puzzle configurations you have to open each Blueprint individually.
- Copying a configuration to a new puzzle requires opening both Blueprints.
- A designer cannot see all puzzles in one place to check for duplicate sequences or missing entries.
- The configuration is tightly coupled to the class hierarchy.

Data assets and data tables solve this by moving configuration out of class defaults and into standalone assets that can be opened, compared, and edited independently of any Blueprint.

---

## UDataAsset

`UDataAsset` is a `UObject` subclass designed to be created as a standalone asset in the Content Browser. It holds data, nothing else. No gameplay logic, no components, no tick.

### Declaring a Data Asset

```cpp
// PuzzleData.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "PuzzleData.generated.h"

UCLASS(BlueprintType)
class ESCAPEROOMLAB_API UPuzzleData : public UDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Puzzle")
    FName PuzzleId;

    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Puzzle")
    FText DisplayName;

    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Puzzle")
    TArray<FName> CorrectSequence;

    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Puzzle")
    FName NarrativeEventOnSolve;

    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Puzzle")
    bool bResetOnFailure = true;
};
```

`BlueprintType` on the `UCLASS` makes the asset referenceable from Blueprint graphs and `EditAnywhere` properties.

### Creating an Asset in the Editor

Once the class is compiled, right-click in the Content Browser and look under **Miscellaneous > Data Asset**. Select `UPuzzleData` from the class picker. Name the asset something descriptive like `DA_SequencePuzzle_01`.

Open it. The Details panel shows all the `EditAnywhere` properties. Fill them in. Save the asset.

### Referencing a Data Asset from C++

On `ASequencePuzzle`, replace the inline properties with a single data asset reference:

```cpp
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Puzzle")
TObjectPtr<UPuzzleData> PuzzleData;
```

In `BeginPlay` or `Activate`, read from it:

```cpp
void ASequencePuzzle::Activate()
{
    Super::Activate();

    if (!PuzzleData)
    {
        UE_LOG(LogTemp, Warning, TEXT("%s has no PuzzleData assigned."), *GetName());
        return;
    }

    // Use PuzzleData->CorrectSequence, PuzzleData->bResetOnFailure, etc.
}
```

The designer now configures the puzzle by assigning a data asset rather than editing class defaults. Multiple puzzles can share the same data asset if they should have the same configuration. The configuration lives in one place.

### Primary Data Assets

For assets that need to be managed by the Asset Manager (loaded on demand, streamed, referenced by soft pointer), subclass `UPrimaryDataAsset` instead of `UDataAsset`. `UPrimaryDataAsset` has a `FPrimaryAssetId` that the Asset Manager uses for asset discovery and async loading.

For the escape room project `UDataAsset` is sufficient. `UPrimaryDataAsset` becomes relevant when you need to stream assets in and out of memory during gameplay, which is a shipping concern rather than a student project concern.

---

## Soft References

A `TObjectPtr<UPuzzleData>` on an actor is a hard reference. When the level loads, the asset loads too. For a small project this is fine.

A soft reference (`TSoftObjectPtr`) stores the asset path but does not load the asset until you explicitly request it. This is important for large projects where loading everything at startup would cause long load times.

```cpp
// Hard reference -- loads with the actor
UPROPERTY(EditAnywhere)
TObjectPtr<UPuzzleData> PuzzleData;

// Soft reference -- loads on demand
UPROPERTY(EditAnywhere)
TSoftObjectPtr<UPuzzleData> PuzzleDataSoft;
```

To load a soft reference:

```cpp
UPuzzleData* Data = PuzzleDataSoft.LoadSynchronous();
```

`LoadSynchronous` blocks until the asset is loaded. For async loading without blocking, use the `FStreamableManager`. For the escape room project, hard references are appropriate since assets are small and level-scoped.

---

## UDataTable

A `UDataTable` is a spreadsheet-like asset where each row is a struct. It is the right tool when you have many items of the same type that a designer might want to edit in a tabular view: key definitions, pickup types, dialogue lines, room configurations.

### Defining the Row Struct

The row struct must inherit from `FTableRowBase`:

```cpp
// ItemData.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataTable.h"
#include "ItemData.generated.h"

USTRUCT(BlueprintType)
struct FItemData : public FTableRowBase
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FText DisplayName;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FText Description;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TObjectPtr<UTexture2D> Icon;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bIsConsumable = false;
};
```

`FTableRowBase` adds the `RowName` column automatically. Each row in the table has a unique `FName` row name that you use to look up entries.

### Creating the Data Table

In the Content Browser, right-click > **Miscellaneous > Data Table**. When prompted for the row struct, select `FItemData`. Name it `DT_Items`.

Open it. The table editor shows columns matching your struct fields and a row per item. Add rows, fill in values, save. You can also import from a CSV file if your designer prefers working in a spreadsheet application.

### Reading from a Data Table in C++

Reference the table asset on an actor or GameMode:

```cpp
UPROPERTY(EditAnywhere, Category = "Data")
TObjectPtr<UDataTable> ItemTable;
```

Look up a row by name:

```cpp
if (ItemTable)
{
    FItemData* Row = ItemTable->FindRow<FItemData>(FName("BrassKey"), TEXT("Item lookup"));
    if (Row)
    {
        UE_LOG(LogTemp, Log, TEXT("Found item: %s"), *Row->DisplayName.ToString());
    }
}
```

The second argument to `FindRow` is a context string used in warning messages if the row is not found. It does not affect functionality.

`FindRow` returns a raw pointer to the struct data inside the table. Do not store this pointer long-term, if the table asset is garbage collected or reloaded, the pointer becomes invalid. Copy the data if you need to keep it:

```cpp
if (FItemData* Row = ItemTable->FindRow<FItemData>(FName("BrassKey"), TEXT("")))
{
    FItemData LocalCopy = *Row;
    // Use LocalCopy safely.
}
```

### Data Table vs Data Asset

Use a **Data Asset** when:
- Each configuration is a standalone thing with its own identity (a specific puzzle, a specific enemy type).
- You want to assign the configuration directly to an actor in the editor.
- The data has nested structures or object references that do not fit a flat row.

Use a **Data Table** when:
- You have many items of the same shape (all keys, all rooms, all dialogue lines).
- A designer wants a spreadsheet-style overview of all entries.
- You want to drive behaviour from a row name (look up a key by its `FName` id).

For the escape room project, puzzle configurations suit data assets (each puzzle is a distinct configured object). Item definitions suit a data table (all items share the same shape and you want a single table showing all of them).

---

## Applying This to the Escape Room

The practical changes for the project:

**Puzzle configuration via data asset:**
- Create `UPuzzleData` as described above.
- On each placed puzzle actor, assign a `DA_` asset instead of setting values directly on the Blueprint.
- In `Activate`, read from `PuzzleData`.

**Item database via data table:**
- Define `FItemData` with at minimum a display name and icon.
- Create `DT_Items` with one row per collectible item in the game.
- When the player picks up an item, look up its display data from the table using the `FName` key already stored in `Inventory`.
- Use the display data to drive the HUD inventory display from block 05.

Neither change is required for a passing grade. Both directly improve the designer workflow and demonstrate understanding of the data/code separation principle that is part of the VG criteria.

---

## What to Have Working by End of Block

At minimum one of the following:

- `UPuzzleData` implemented and assigned to at least one puzzle in the level. The puzzle reads its configuration from the asset at runtime.
- `FItemData` and `DT_Items` implemented with at least two item entries. The HUD reads display names from the table when updating the inventory display.
