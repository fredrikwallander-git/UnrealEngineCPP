# UnrealCPP

Course material for the Spelprogrammering C++ section at Forsbergs skola.

## Minimum requirements (G)
All of the following are present and working:

* A first-person playable character implemented in C++ (movement and look using Enhanced Input).
* An interaction system implemented in C++. This must be a reusable system, not hard-coded checks. The system should use either a C++ interface or a base interactable component. A door, a button, and a pickup must all use the same system.
* At least two interconnected puzzles, where solving puzzle A enables or affects puzzle B in some way. Each puzzle must involve C++ logic (a state machine, an unlock condition, an item-combination check, etc.), not just BP triggers wired together.
* A narrative trigger system in C++ that fires events based on player actions: at minimum: entering a volume, interacting with an object, completing a puzzle. The narrative reaction can be audio lines, on-screen text, environmental changes, or a combination.
* At least one C++ class exposed to Blueprint in a meaningful way: either a base C++ class with BP children for content variation (e.g. AInteractableBase with BP children for specific puzzle pieces), or BP-callable functions on a C++ subsystem.
* A win/end state that the player can actually reach.
* Source available in a GitHub repository with Git LFS tracking.

## Excellent (VG)
All G requirements are met, plus:

* Three interconnected puzzles with at least one puzzle whose solution depends on combining or sequencing earlier puzzle results.
* Branching or reactive narrative: the narrative system must respond differently based on player choices, order of actions, or failure states. Not just "play sound when X happens", the system must track state and react accordingly. This must be demonstrable by playing through the same section twice and getting different reactions.
* Clean C++/Blueprint boundary: data and content live in BP (puzzle configurations, narrative lines, tuning values), behaviour and architecture live in C++. The group can articulate why each piece lives where it does.