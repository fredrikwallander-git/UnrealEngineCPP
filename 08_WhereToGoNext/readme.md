# 08 - Where to Go Next

This block does not introduce new project requirements. The escape room course material is complete. What follows here is a map of the territory beyond this course: systems you will encounter in professional Unreal development, concepts worth understanding and directions for independent study.

---

## Gameplay Ability System (GAS)

GAS is Epic's production framework for ability-based gameplay. It is used in Fortnite, Lyra, and many other shipped titles. It covers everything this course built manually: state machines, status effects, attribute sets, cooldowns, and event-driven ability triggering, all in a networked, data-driven package.

The core concepts:

**Ability System Component (ASC):** Added to an actor (usually the character or the player state). Acts as the central manager for all abilities and attributes on that actor.

**Gameplay Ability:** A discrete action the actor can perform. Activating an ability, attacking, dashing, interacting. Each ability is a C++ or Blueprint class that defines what happens, under what conditions it can activate, and what it costs.

**Gameplay Effect:** A change applied to an actor's attributes. Damage, healing, speed buffs, stun. Effects can be instant, periodic, or duration-based. They are data assets, not code.

**Gameplay Attribute Set:** A struct of numeric attributes (health, stamina, mana) with built-in replication and clamping.

**Gameplay Tags:** A hierarchical tag system (`Combat.Debuff.Stun`, `Ability.Active`, `State.Dead`) used to drive conditions, queries, and filtering across the entire ability system. Replaces boolean flags with a queryable tag container.

Why not use GAS for this course?<br>
It has a steep setup cost. Understanding it requires solid C++ fundamentals, event-driven architecture knowledge, and comfort with Unreal's replication system. This course built those fundamentals. GAS is the natural next step if you are heading toward multiplayer action games or RPGs.

Where to start: the Lyra Starter Game (available in the Epic Launcher) is Epic's reference implementation of GAS in a production context. The community-written GAS documentation by tranek on GitHub (`tranek/GASDocumentation`) is the most complete reference available.

---

## Multiplayer and Replication

Everything built in this course assumes a single-player context. In multiplayer, game state lives on a server and is selectively replicated to clients. The systems you have built need adjustment before they work correctly in a networked game.

Key concepts to understand:

**Authority:** Only the server has authority over gameplay state. Clients send input to the server. The server processes it and replicates the result. Code that changes game state should check `HasAuthority()` before running.

**Replication:** Variables marked `Replicated` are synchronized from server to clients automatically. Functions marked `Server` or `Client` run on specific machines. `NetMulticast` functions run on all machines.

**RPCs (Remote Procedure Calls):** The mechanism for running a function on a different machine. A client calls a `Server` RPC to ask the server to do something. The server calls a `NetMulticast` RPC to tell all clients something happened.

**Relevancy and Priority:** Not every actor is replicated to every client. Actors outside the client's relevancy range stop being updated. This affects how you structure always-important state (put it in `GameState`) versus per-player state (put it in `PlayerState`).

The escape room puzzle system as built in block 04 would need authority checks on `SetState` and replication on `CurrentState` to work correctly in multiplayer. The narrative director would need to use multicast RPCs to fire events on all clients simultaneously.

Where to start: Epic's Multiplayer Network Compendium by Cedric Neukirchen is the most cited introduction. The Tom Looman Unreal Engine 5 course covers networked gameplay in depth.

---

## Entity Component System (ECS) and Mass Entity

Traditional Unreal Actor/Component architecture is object-oriented: each actor is an object with state and behaviour. ECS inverts this. Entities are just identifiers. Components are plain data structs. Systems process components in bulk with no virtual dispatch and excellent cache performance.

Unreal's Mass Entity framework (introduced in UE5, used in The Matrix Awakens demo) is an ECS implementation built into the engine. It is designed for simulating thousands of entities simultaneously: crowds, traffic, projectiles, AI agents.

For the escape room project Mass Entity is not appropriate. But if you are interested in performance-critical simulation (a city builder, a crowd system, a swarm AI), understanding ECS fundamentally changes how you think about game architecture.

Where to start: read about Data-Oriented Design (DOD) before looking at Mass Entity. Then look at Epic's Mass Entity documentation and the CitySample project.

---

## Procedural Content Generation (PCG)

PCG is Unreal 5.2's framework for generating content algorithmically at edit time or runtime. Scatter foliage, generate dungeon layouts, populate environments with rules rather than hand-placement.

For a game like an escape room, PCG could generate room layouts procedurally, place props based on rules, or vary puzzle configurations between runs. It is entirely node-based but has a C++ API for writing custom PCG nodes.

Where to start: Epic's PCG documentation and the Electric Dreams environment demo (available on the Marketplace) which is built almost entirely with PCG.

---

## Niagara

Niagara is Unreal's particle and visual effects system. Understanding it at a C++ level means writing custom Niagara data interfaces: C++ classes that expose game data (positions, velocities, gameplay state) to Niagara emitters at runtime.

For a VFX programmer role, knowing how to write a Niagara data interface that reads from your gameplay systems is a significant differentiator.

Where to start: build several effects in the Niagara editor first to understand how emitters, modules, and parameters work. Then look at the `UNiagaraDataInterface` class in the engine source.

---

## Shader Development and HLSL

If you are drawn toward technical art or graphics programming, Unreal exposes shader authoring through Material graphs and through custom HLSL nodes. Writing a compute shader in Unreal requires using the Rendering Hardware Interface (RHI) layer, which is significantly more complex than gameplay C++.

Understanding the rendering pipeline, how draw calls are batched, how materials affect performance, and how to profile GPU costs in RenderDoc are all skills that bridge gameplay programming and graphics programming.

Where to start: the Unreal Rendering documentation, Ben UI's material and shader tutorials, and the `GlobalShader` plugin example in the engine source.

---

## Save and Load

This course did not cover saving game state. In a shipped product you need it. Unreal provides `USaveGame`, a simple serialization system for saving arbitrary data to disk.

The pattern:

```cpp
// Save
UMySaveGame* SaveData = Cast<UMySaveGame>(
    UGameplayStatics::CreateSaveGameObject(UMySaveGame::StaticClass()));
SaveData->SolvedPuzzles = SolvedPuzzleIds;
UGameplayStatics::SaveGameToSlot(SaveData, TEXT("Slot1"), 0);

// Load
UMySaveGame* SaveData = Cast<UMySaveGame>(
    UGameplayStatics::LoadGameFromSlot(TEXT("Slot1"), 0));
if (SaveData)
    SolvedPuzzleIds = SaveData->SolvedPuzzles;
```

For the escape room, saving means serializing which puzzles are solved, the inventory contents, and the door states. The puzzle state machine from block 04 already has `CurrentState` per puzzle. A save system reads that state out and writes it back on load.

---

## Animation: Animation Blueprint and Control Rig

The escape room used a timeline to rotate a door. Professional character animation is driven by the Animation Blueprint system: a state machine that blends animation assets based on gameplay state variables. Speed, direction, whether the character is grounded, whether they are aiming, all feed into the animation graph.

Control Rig adds procedural animation on top: inverse kinematics (IK) for foot placement on uneven terrain, hand placement on surfaces, head tracking. It is a node-based rigging system that compiles to runtime code.

For a gameplay programmer, understanding how to expose variables from C++ to the Animation Blueprint (`GetVelocity`, `bIsJumping`, aiming angle) and how to trigger animation montages from code is essential. Character animation is one of the most visible parts of any third-person game.

Where to start: the Third Person template's Animation Blueprint is a good starting point. It already drives blend spaces and transitions from the character's velocity. Read through it and trace how each variable connects to the animation graph.

---

## AI: Behaviour Trees and EQS Beyond the Basics

The AI Programming course covered Behaviour Trees and NavMesh. Going deeper:

**Environment Query System (EQS):** Spatial queries that find optimal positions in the world. "Find the best cover position given enemy locations and the player's position." EQS runs a scored query across candidate points and returns the best result. It is the standard tool for AI positioning in Unreal.

**Mass AI:** Built on Mass Entity, designed for simulating large numbers of AI agents efficiently. Uses a Zonegraph for navigation instead of NavMesh, which scales to thousands of agents where NavMesh becomes a bottleneck.

**Smart Objects:** Actors in the world that advertise available interactions to AI agents. A bench advertises "sit." A workbench advertises "work." AI agents query for available smart objects and claim them. Decouples AI behaviour from specific actor types.

Where to start: extend the AI project from the AI Programming course with EQS cover queries. Then look at Smart Objects in the Unreal documentation.

---

## Profiling and Optimisation

Knowing how to build a feature is half the job. Knowing why it runs slowly and how to fix it is the other half.

Tools to know:

**Unreal Insights:** the primary profiling tool for CPU and GPU performance in Unreal. Records traces, shows thread timelines, identifies hot spots.

**Stat commands:** `stat fps`, `stat unit`, `stat game`, `stat gpu` give real-time performance data in the viewport without opening a full profiler.

**RenderDoc:** GPU frame debugger for diagnosing rendering issues, overdraw, and shader performance.

**Tracy:** External CPU profiler with very low overhead, commonly used in game studios alongside Unreal Insights.

**Asset Auditor:** Shows asset sizes, references, and load times. Identifies unnecessarily large textures and assets loaded when they should not be.

Common performance traps in Unreal C++:

Calling `GetAllActorsOfClass` in Tick. Firing delegates that trigger Blueprint graphs every frame. Using `FindComponentByClass` in Tick on many actors. Creating and destroying actors frequently instead of pooling. Loading assets synchronously on the game thread.

None of these are catastrophic at the scale of a student project. They become problems at shipping scale. Recognizing them now means not writing them into a production codebase later.

---

## Recommended Resources

**Documentation and reference:**
- Unreal Engine documentation: docs.unrealengine.com

**Communities:**
- Unreal Source Discord: the most active community for Unreal C++ questions
- r/unrealengine: good for general questions, variable quality of answers
- Unreal Engine forums (forums.unrealengine.com): slower but often has answers from Epic engineers

---

## Final Note

The systems you built in blocks 01 through 07 are the same systems that appear in shipped games, just at smaller scale. A state machine is a state machine whether it has four states or forty. A delegate is a delegate whether it connects two actors or two hundred. The C++ patterns you learned here compose into everything else.

The gap between a student project and a shipped product is not which features exist. It is the quality of the code, the robustness of the edge case handling, the performance at scale, and the clarity of the architecture to someone reading it for the first time. Those things improve with practice and with reading other people's good code.

