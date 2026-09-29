Entity AI Architecture Specification

C++ Engine + Embedded LuaJIT

V1 → V4

---

1. Purpose

This specification defines the architecture for autonomous entities in the engine.

The AI system is a native C++ subsystem. Embedded LuaJIT is the extension, configuration, behavior-authoring, and modding layer. It is not responsible for high-frequency core simulation.

The architecture must support:

- scripted entities;
- finite-state behavior;
- behavior graphs;
- utility-based decision making;
- neural cognition where useful;
- hybrid behavioral/neural cognition;
- large entity populations;
- AI level-of-detail;
- deterministic or controlled simulation;
- CPU and future GPU execution;
- terrain-aware entities;
- physically embodied entities;
- mod-defined AI behavior;
- future cognition implementations without redesigning the entity system.

The architecture must prevent AI implementations from becoming coupled to voxel storage, meshing, rendering, GPU geometry allocation, or any particular cognition implementation.

---

2. Core Architectural Rule

The fundamental rule is:

«AI observes the world and produces intent. It does not directly manipulate authoritative world state.»

Primary flow:

World State
    |
    v
Perception
    |
    v
Memory
    |
    v
Cognition
    |
    v
Goal Selection
    |
    v
Navigation
    |
    v
Motor Intent
    |
    v
Entity Controller
    |
    v
Physics / Simulation
    |
    v
World State

AI must not directly:

- edit voxel storage;
- invoke mesh generation;
- allocate GPU geometry;
- modify renderer state;
- manipulate VGP internals;
- bypass authoritative physics;
- directly overwrite authoritative entity transforms.

This establishes a closed simulation loop:

WORLD -> SENSE -> THINK -> ACT -> WORLD

---

3. High-Level Architecture

                         ENGINE
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
     World State        Physics            VGP
          |                |                |
          +--------+-------+                |
                   |                        |
                   v                        |
            AIWorldInterface                |
                   |                        |
                   v                        |
              Entity AI                     |
                   |                        |
       +-----------+-----------+            |
       |           |           |            |
       v           v           v            |
  Perception     Memory     Scheduler        |
       |           |           |            |
       +-----------+-----------+            |
                   |                        |
                   v                        |
             Cognition API                  |
                   |                        |
       +-----------+-----------+            |
       |           |           |            |
       v           v           v            |
    State       Utility      Neural          |
   Machine       Model       Model           |
       |           |           |            |
       +-----------+-----------+            |
                   |                        |
                   v                        |
              Goal System                   |
                   |                        |
                   v                        |
              Navigation                    |
                   |                        |
                   v                        |
              MotorIntent                   |
                   |                        |
                   v                        |
           Entity Controller                |
                   |                        |
                   v                        |
                Physics                     |
                   |                        |
                   v                        |
              World State ------------------+

VGP and AI are sibling engine systems.

AI does not sit inside VGP.

VGP does not sit inside AI.

Both operate against shared authoritative engine state through defined interfaces.

---

4. Authoritative World Boundary

AI perception should primarily query authoritative world, entity, physics, and spatial data.

It must not depend on rendered geometry being available.

This distinction is important.

The following must remain true:

Chunk not rendered
        !=
Chunk nonexistent to AI

Likewise:

Object culled by renderer
        !=
Object invisible to simulation

and:

GPU geometry not resident
        !=
Terrain unavailable to AI

Therefore:

Authoritative World State
          |
          v
   AIWorldInterface
          |
          v
      Perception

is the primary AI path.

VGP may later provide optimized spatial or GPU-derived services, but these are accelerators rather than the source of simulation truth.

---

5. Relationship to VGP

VGP remains the Voxel Geometry Pipeline.

Its responsibilities remain centered on:

Voxel / World Data
       |
       v
      VGP
       |
       +-- geometry generation
       +-- meshing
       +-- geometry allocation
       +-- visibility / culling
       +-- geometry LOD
       +-- indirect rendering
       +-- GPU-driven geometry systems

AI instead operates through:

World / Entity / Physics State
            |
            v
     AIWorldInterface
            |
            v
           AI

Future acceleration may permit:

VGP spatial structures
        |
        v
AI query acceleration

but AI behavior must remain correct without those optimizations.

---

6. AIWorldInterface

AI requires a stable world-query abstraction.

Conceptually:

class AIWorldInterface
{
public:
    VisionResult query_vision(
        EntityID entity,
        const VisionQuery& query) const;

    AudioResult query_audio(
        EntityID entity,
        const AudioQuery& query) const;

    ProximityResult query_entities(
        EntityID entity,
        const ProximityQuery& query) const;

    TerrainResult query_terrain(
        EntityID entity,
        const TerrainQuery& query) const;

    NavigationResult query_navigation(
        EntityID entity,
        const NavigationQuery& query) const;
};

The implementation may internally use:

- authoritative world storage;
- entity spatial indices;
- physics queries;
- voxel occupancy;
- navigation structures;
- spatial hierarchies;
- future GPU query systems.

Cognition does not know which implementation produced the observation.

---

7. Entity AI Composition

An AI-controlled entity may reference:

EntityAI
 |
 +-- PerceptionState
 +-- MemoryState
 +-- CognitiveModel
 +-- GoalState
 +-- NavigationState
 +-- MotorIntent
 +-- SchedulerState
 +-- AILODState

These should normally be handles or indices into pooled storage rather than individually heap-allocated object graphs.

Prefer:

EntityID -> AIHandle -> pooled AI data

over:

Entity
 |
 +-- new Sensor
 +-- new Memory
 +-- new Brain
 +-- new Navigation
 +-- new Behavior

The former scales substantially better.

---

8. Perception System

Perception converts world state into entity-specific observations.

Initial sensory channels:

Vision
Hearing
Proximity
Terrain
Touch
Internal State

Future channels can include:

Smell
Temperature
Light
Vibration
Chemical Fields
Social Signals

Example:

struct SensoryState
{
    VisionResult vision;
    AudioResult audio;
    ProximityResult proximity;
    TerrainResult terrain;

    float hunger;
    float thirst;
    float fatigue;
    float pain;
};

The perception system performs filtering.

The cognitive system should not receive every entity, voxel, collision object, or sound in the world.

---

9. Perception Pipeline

The preferred pipeline is:

World Query
    |
    v
Broad Phase
    |
    v
Candidate Set
    |
    v
Entity-specific filtering
    |
    v
Sensory observation
    |
    v
SensoryState

For vision, for example:

Spatial query
    |
    v
Nearby candidates
    |
    v
FOV filtering
    |
    v
Occlusion testing
    |
    v
Visibility result

This becomes increasingly important as population size grows.

---

10. Memory

Memory is independent from cognition.

An entity may therefore retain memory regardless of whether it uses:

- scripted logic;
- state machines;
- utility AI;
- neural cognition;
- hybrid cognition.

Memory categories should support:

Working Memory
Episodic Memory
Spatial Memory
Social Memory
Knowledge

Example:

struct EntityMemory
{
    EntityID last_threat;
    EntityID last_food_source;

    Vec3 last_known_threat_position;

    float threat_confidence;
};

Large variable-length records should live in pooled storage rather than directly inside every entity.

---

11. Memory Budgeting

Memory must have bounded costs.

Potential mechanisms include:

maximum records
        +
expiration
        +
importance
        +
recency
        +
AI LOD

A distant simulated animal does not need the same memory fidelity as a high-priority nearby NPC.

Memory itself can therefore participate in simulation LOD.

---

12. Cognition Interface

Cognition must be replaceable.

Conceptually:

class CognitiveModel
{
public:
    virtual ~CognitiveModel() = default;

    virtual void reset() = 0;

    virtual void stimulate(
        const SensoryState& sensory,
        const EntityMemory& memory) = 0;

    virtual void update(float dt) = 0;

    virtual CognitiveOutput output() const = 0;
};

Supported implementations can include:

ScriptedModel
StateMachineModel
BehaviorGraphModel
UtilityModel
NeuralModel
HybridModel

No surrounding entity system should depend on one particular implementation.

This is the architectural seam that allows additional cognition systems to be introduced later without restructuring entities.

---

13. Cognitive Output

Cognition should generally produce semantic drives, decisions, or goal candidates.

Example:

struct CognitiveOutput
{
    float hunger_drive;
    float fear_drive;
    float curiosity_drive;
    float social_drive;

    EntityID target;
    GoalType preferred_goal;
};

Example output:

fear        = 0.91
hunger      = 0.24
curiosity   = 0.08
target      = Entity 823
goal        = Escape

Cognition does not need to know how to physically escape.

That belongs downstream.

---

14. Goal System

The goal system converts cognition into explicit objectives.

Example:

Cognition
    |
    | fear = 0.91
    | target = predator
    v
Goal Selection
    |
    v
Escape predator

Potential goals include:

Idle
Explore
Travel
Follow
Flee
Attack
Defend
Eat
Drink
Rest
Interact
Investigate
ReturnHome

Mods can add additional goal types through controlled extension mechanisms.

---

15. Navigation

Navigation converts goals into spatial solutions.

Goal
 |
 v
Navigation Request
 |
 v
Route / Movement Target
 |
 v
Locomotion

Navigation can eventually support different movement domains:

Ground
Swimming
Flying
Climbing
Burrowing
Mixed traversal

The cognition model does not need to understand the pathfinding algorithm.

---

16. MotorIntent

AI ultimately produces intent rather than authoritative movement.

Example:

struct MotorIntent
{
    float move_forward;
    float move_right;

    float turn;
    float look;

    float jump;

    float attack;
    float defend;
    float interact;
};

Example:

move_forward = 0.87
move_right   = 0.10
turn         = -0.31
attack       = 0.00

MotorIntent is consumed by the entity controller.

---

17. Entity Controller

The controller translates abstract intent into valid physical behavior.

MotorIntent
     |
     v
Entity Controller
     |
     v
Locomotion
     |
     v
Physics
     |
     v
World State

The controller owns:

- acceleration constraints;
- maximum speed;
- turning behavior;
- movement modes;
- jump validity;
- physical interaction;
- collision response integration;
- animation-facing movement state.

This separates cognition from embodiment.

A wolf and bird can therefore share cognition concepts while using completely different controllers.

---

18. Closed-Loop Entity Simulation

The resulting system forms a closed feedback loop:

              +----------------------+
              |                      |
              v                      |
          Perception                  |
              |                      |
              v                      |
            Memory                    |
              |                      |
              v                      |
          Cognition                   |
              |                      |
              v                      |
             Goal                     |
              |                      |
              v                      |
          Navigation                  |
              |                      |
              v                      |
         MotorIntent                  |
              |                      |
              v                      |
           Controller                 |
              |                      |
              v                      |
            Physics                   |
              |                      |
              v                      |
             World -------------------+

An entity therefore experiences the consequences of its own actions.

---

19. AI Scheduler

AI must not automatically execute every subsystem for every entity every frame.

The scheduler determines:

Does this entity update?
When does perception update?
When does cognition update?
When does navigation update?
When does memory maintenance run?
Which cognition backend runs?
What simulation fidelity is required?

The scheduler therefore becomes a central scalability component.

---

20. Independent Simulation LOD

Do not bind all LOD systems together.

Maintain conceptually independent:

Rendering LOD
Geometry LOD
Physics LOD
AI LOD
Simulation LOD

These can influence one another, but they are not equivalent.

For example:

Entity A:
far from camera
important to simulation

Rendering LOD = low
AI LOD        = medium

while:

Entity B:
visible nearby decorative animal

Rendering LOD = high
AI LOD        = low

---

21. AI LOD

Recommended initial levels:

AI LOD 0
Dormant / statistical

AI LOD 1
Coarse simulation

AI LOD 2
Normal behavioral simulation

AI LOD 3
High-fidelity cognition

AI LOD 4
Specialized expensive cognition

Crucially, these describe simulation cost, not specific AI technologies.

A future implementation can change what each level means without changing the architecture.

---

22. Update Frequencies

Subsystems should operate at different frequencies.

Initial configurable targets might resemble:

Motor / controller:
simulation tick

Perception:
5-30 Hz depending on sensor

Behavior:
10-30 Hz

Navigation:
event-driven + periodic validation

Memory:
event-driven + maintenance

Expensive cognition:
independent schedule

Dormant entities:
coarse/event-driven simulation

These are starting ranges for profiling, not permanent constants.

---

23. Event-Driven AI

Not every AI operation needs polling.

Prefer events where appropriate:

damage received
entity entered sensor volume
sound emitted
goal completed
path invalidated
terrain changed
resource discovered
entity died
relationship changed

Then:

Engine Event
    |
    v
AI Event Queue
    |
    v
Relevant Entities

This can eliminate substantial unnecessary work.

---

24. Behavior Models

V1 should establish conventional behavior first.

Initial implementations:

StateMachineModel
UtilityModel

Then optionally:

BehaviorGraphModel

The system should support combinations such as:

State Machine
      |
      v
Utility Selection
      |
      v
Goal

or:

Utility Model
      |
      v
Behavior Graph
      |
      v
Goal

No single AI methodology needs to control the whole stack.

---

25. Neural Cognition

Neural cognition remains useful as an optional engine capability, but it should be an engine-native generalized model, not tied to a biological simulation.

The useful adopted pattern is:

Sensory Input
     |
     v
Neural Processing
     |
     v
Semantic Outputs
     |
     v
Behavior / Utility
     |
     v
Goal

Initial neural models should be deliberately small and task-specific.

Example development scales:

64 units
256 units
1024 units

These are test scales rather than architectural limits.

---

26. Neural Runtime

If neural cognition proves useful, implement:

NeuralRuntime
 |
 +-- UnitStorage
 +-- ConnectionStorage
 +-- InputBuffer
 +-- StateBuffer
 +-- OutputBuffer
 +-- Scheduler
 +-- Backend

Backends:

CPU first
GPU later if profiling justifies it

GPU implementation must not precede a verified CPU reference.

---

27. Neural Storage

Avoid per-entity heap-heavy graphs.

Avoid:

struct Entity
{
    std::vector<Neuron> neurons;
    std::vector<Connection> connections;
};

Prefer:

Global Neural Storage
 |
 +-- unit pool
 +-- connection pool
 +-- state buffers
 +-- input buffers
 +-- output buffers
 +-- model descriptors

Entity:

struct NeuralHandle
{
    uint32_t unit_offset;
    uint32_t unit_count;

    uint32_t connection_offset;
    uint32_t connection_count;
};

This enables batching and future accelerator execution.

---

28. Hybrid Cognition

The preferred advanced architecture is hybrid rather than forcing neural computation to perform every task.

Example:

Perception
    |
    v
Neural Model
    |
    v
Internal Drives
    |
    v
Utility Model
    |
    v
Goal
    |
    v
Navigation
    |
    v
MotorIntent

Example:

Neural output:
fear   = 0.91
hunger = 0.32

Utility evaluation:
flee   = 0.94
hunt   = 0.27

Selected goal:
Escape

Navigation:
Find escape route

Controller:
Execute movement

This provides emergent input processing without sacrificing controllability and debugging.

---

29. Future Cognition Extensibility

No specific experimental cognition technology needs to exist in the roadmap.

Instead, preserve this seam:

CognitiveModel
 |
 +-- Current implementations
 |
 +-- Future implementations

Any future model only needs to satisfy the cognition contract:

SensoryState + Memory
          |
          v
   CognitiveModel
          |
          v
   CognitiveOutput

This is sufficient future-proofing.

No engine work should be performed now solely to support hypothetical future cognition systems.

---

30. LuaJIT Boundary

LuaJIT provides extensibility.

Lua may define:

Entity definitions
Sensor configuration
Behavior configuration
Utility curves
Goals
Decision rules
AI parameters
Mod-specific behavior
Cognition configuration

C++ owns:

Entity storage
AI scheduler
Perception execution
Memory storage
High-frequency cognition
Navigation runtime
Physics
Neural runtime
GPU buffers
VGP internals

---

31. LuaJIT Example

A mod could define:

WolfAI = {
    sensors = {
        vision = true,
        hearing = true,
        smell = true
    },

    cognition = {
        model = "wolf_standard"
    },

    behaviors = {
        hunt = true,
        flee = true,
        follow_pack = true
    }
}

C++ loads and validates the configuration.

Lua describes behavior.

C++ executes the expensive simulation.

---

32. Lua API Philosophy

Expose semantic operations.

Good examples:

entity:get_position()
entity:get_velocity()

memory:remember(...)
memory:recall(...)

ai:set_goal(...)
ai:get_goal(...)

perception:get_visible_entities()
perception:get_last_sound()

Do not expose raw ownership internals such as:

voxel storage pointers
geometry heap pointers
GPU command buffers
neural storage pointers
physics internal pointers
allocator internals

Lua must operate through controlled engine APIs.

---

33. Recommended Module Layout

engine/
|
+-- ai/
    |
    +-- core/
    |   +-- entity_ai.h
    |   +-- cognitive_model.h
    |   +-- cognitive_output.h
    |   +-- motor_intent.h
    |   +-- ai_types.h
    |
    +-- perception/
    |   +-- perception_system.h
    |   +-- sensory_state.h
    |   +-- vision.h
    |   +-- audio.h
    |   +-- proximity.h
    |   +-- terrain_sense.h
    |
    +-- memory/
    |   +-- memory_system.h
    |   +-- working_memory.h
    |   +-- episodic_memory.h
    |   +-- spatial_memory.h
    |
    +-- behavior/
    |   +-- state_machine.h
    |   +-- behavior_graph.h
    |   +-- utility_model.h
    |
    +-- goals/
    |   +-- goal.h
    |   +-- goal_system.h
    |
    +-- navigation/
    |   +-- navigation_system.h
    |   +-- path.h
    |   +-- locomotion.h
    |
    +-- scheduler/
    |   +-- ai_scheduler.h
    |   +-- ai_lod.h
    |   +-- update_budget.h
    |
    +-- neural/
    |   +-- neural_model.h
    |   +-- neural_runtime.h
    |   +-- neural_storage.h
    |   +-- neural_backend.h
    |   +-- cpu_backend.h
    |   +-- gpu_backend.h
    |
    +-- scripting/
        +-- ai_lua_api.h
        +-- ai_bindings.cpp

The "neural/" module should remain optional until V3.

---

34. Ownership Contract

Subsystem ownership should remain explicit.

Responsibility| Owner
Authoritative voxel/world data| World subsystem
Entity identity/state| Entity runtime
Collision/physical movement| Physics
Geometry generation| VGP
Geometry allocation| VGP
Rendering visibility| VGP/renderer
AI world queries| AIWorldInterface
Sensory filtering| Perception
AI memories| Memory system
Decision processing| CognitiveModel
Objective selection| Goal system
Route selection| Navigation
Movement request| MotorIntent
Physical execution| Entity controller/physics
AI scheduling| AI scheduler
AI configuration/modding| LuaJIT interface
High-frequency AI computation| C++ runtime

No subsystem should silently assume ownership belonging to another.

---

35. V1 - Entity Intelligence Foundation

V1 establishes the complete functional pipeline without advanced neural processing.

Implement:

EntityAI
AIWorldInterface
PerceptionSystem
SensoryState
MemorySystem
CognitiveModel
StateMachineModel
UtilityModel
CognitiveOutput
GoalSystem
Navigation interface
MotorIntent
Entity Controller interface
AI Scheduler
Lua AI configuration

Required pipeline:

World
  |
  v
Perception
  |
  v
Memory
  |
  v
Behavior / Utility
  |
  v
Goal
  |
  v
Navigation
  |
  v
MotorIntent
  |
  v
Controller
  |
  v
Physics

V1 Acceptance Gate

V1 is complete when an entity can:

1. perceive relevant world information;
2. retain useful state;
3. choose among multiple behaviors;
4. select a goal;
5. request navigation;
6. generate MotorIntent;
7. physically act through the normal controller/physics path;
8. react to changed world conditions;
9. be configured through LuaJIT;
10. run without renderer or VGP-specific AI dependencies.

---

36. V2 - Population Scale

V2 makes the V1 architecture scalable.

Implement:

AI LOD
sleep / wake
batched perception
pooled AI storage
pooled memory
event-driven AI
update budgets
priority scheduling
population simulation
coarse distant simulation

Add:

AI LOD 0
AI LOD 1
AI LOD 2

Benchmark increasing entity populations.

Measure separately:

perception time
cognition time
navigation time
memory time
scheduler time
total AI frame budget

V2 Acceptance Gate

V2 is complete when:

- AI work obeys a configurable time budget;
- low-priority entities can degrade gracefully;
- dormant entities consume minimal processing;
- perception can execute in batches;
- AI storage does not require excessive per-entity heap allocation;
- large populations can be profiled deterministically;
- entity behavior remains functionally correct while moving between AI LOD levels.

---

37. V3 - Advanced Cognition

V3 adds optional neural and hybrid cognition.

Implement:

NeuralModel
NeuralRuntime
pooled neural storage
input channels
output channels
CPU reference backend
hybrid neural + utility behavior
AI LOD 3

Pipeline:

SensoryState
     |
     v
NeuralModel
     |
     v
Semantic Drives
     |
     v
Utility / Behavior
     |
     v
Goal
     |
     v
MotorIntent

GPU work occurs only if profiling demonstrates that CPU neural execution is a meaningful bottleneck.

V3 Acceptance Gate

V3 is complete when:

- neural models implement the same CognitiveModel contract;
- conventional entities require no neural runtime;
- neural inputs use semantic sensory channels;
- neural outputs use semantic cognition channels;
- multiple neural entities can be processed through pooled storage;
- hybrid cognition works;
- neural entities can transition through AI LOD safely;
- the CPU implementation has deterministic test fixtures where practical.

---

38. V4 - Large-Scale Adaptive AI

V4 focuses on scale and deeper integration rather than introducing a specific cognition technology.

Potential work includes:

GPU cognition backend
GPU-batched perception
hierarchical navigation
large-scale spatial queries
advanced memory policies
social/group AI
pack/herd behavior
shared knowledge
adaptive behavior
world-scale population simulation
advanced AI LOD

Potential architecture:

                   AI Scheduler
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
      Perception     Cognition    Navigation
          |             |             |
          +-------------+-------------+
                        |
                        v
                    Entity AI
                        |
                        v
                   MotorIntent
                        |
                        v
                     Physics

V4 Acceptance Gate

V4 is complete when expensive AI workloads can scale across large populations without changing the entity-facing AI contract.

Any GPU acceleration must remain replaceable by a CPU implementation for testing and fallback where practical.

---

39. Relationship to the VGP V1-V4 Roadmap

The two systems progress in parallel.

ENGINE V1
|
+-- VGP
|   +-- CPU geometry pipeline
|   +-- Naive Face Mesher
|   +-- Greedy Mesher
|   +-- persistent geometry allocation
|
+-- AI
    +-- perception
    +-- memory
    +-- conventional cognition
    +-- goals
    +-- navigation interface
    +-- MotorIntent

ENGINE V2
|
+-- VGP
|   +-- GPU visibility
|   +-- indirect rendering
|   +-- allocator scaling
|
+-- AI
    +-- scheduler scaling
    +-- AI LOD
    +-- batching
    +-- event-driven simulation

ENGINE V3
|
+-- VGP
|   +-- hierarchy
|   +-- geometry LOD
|   +-- streaming
|   +-- advanced meshers
|
+-- AI
    +-- optional neural cognition
    +-- hybrid cognition
    +-- advanced population simulation

ENGINE V4
|
+-- VGP
|   +-- mature GPU-driven geometry
|   +-- advanced traversal
|   +-- specialized terrain pipelines
|
+-- AI
    +-- large-scale adaptive simulation
    +-- advanced perception
    +-- group/social systems
    +-- optional GPU cognition

They share engine infrastructure but remain separate subsystems.

---

40. Mesher Independence

AI must remain independent from:

Naive Face Mesher
Greedy Mesher
Surface Mesher
Transvoxel
LOD Mesher
Special-purpose Terrain Mesher

The existing mesher progression remains:

V1:
Naive Face Mesher
Greedy Mesher
common IMesher -> MeshData contract

V3:
LOD Mesher
Surface / Transvoxel systems

V4:
Special-purpose terrain meshers

AI queries authoritative terrain/world representations rather than asking a mesher for world meaning.

---

41. Allocator Independence

The VGP geometry allocator remains completely outside AI ownership.

AI may request semantic information such as:

terrain occupancy
surface direction
terrain clearance
entity proximity
line of sight
navigation feasibility

AI must not manipulate:

geometry heap
mesh allocation
GPU geometry handles
indirect draw buffers
render command buffers

This separation is mandatory.

---

42. Determinism

The architecture should support controlled deterministic simulation where required.

Potential nondeterministic sources include:

parallel scheduling
floating-point behavior
unordered query results
random behavior
GPU execution
event ordering

Control:

simulation seed
entity ordering
query ordering
AI timestep
random streams
event ordering

Exact GPU/CPU floating-point identity should not be assumed.

Instead, tests should define appropriate tolerances where necessary.

---

43. Testing

Every AI subsystem should be testable without the renderer.

Unit tests:

Perception
Memory
State Machine
Utility
Goal Selection
Navigation
Motor conversion
Scheduler
AI LOD
Neural processing

Simulation tests:

entity detects target
entity remembers target
entity chooses appropriate goal
entity navigates
entity moves
world changes
entity perceives change
entity changes behavior

Scale tests:

100 entities
1,000 entities
10,000 entities
increasing until budget failure

Do not define success by an arbitrary population count.

Define success by measured frame/simulation budgets.

---

44. AI Debugging

Every active entity should eventually expose inspection information.

Example:

Entity: 1234

AI LOD:
2

Cognition:
UtilityModel

Current Goal:
Flee

Target:
Entity 829

Perception:
Threat detected

Memory:
Last threat position available

Drives:
Fear      0.91
Hunger    0.23

Navigation:
Escape route active

Motor:
Forward   0.87
Turn     -0.31

For neural/hybrid entities:

Cognition:
HybridModel

Inputs:
Threat    0.92
Food      0.17

Neural outputs:
Fear      0.88
Approach  0.11

Utility result:
Flee      0.94

Goal:
Escape

Observability is a required feature, not an optional debugging luxury.

---

45. Implementation Order

The implementation sequence is:

01. AI core types
02. MotorIntent
03. AIWorldInterface
04. Perception
05. Memory
06. CognitiveModel contract
07. StateMachineModel
08. UtilityModel
09. Goal system
10. Navigation interface
11. Entity controller integration
12. LuaJIT AI configuration
13. AI scheduler
14. AI LOD
15. Event-driven AI
16. Pooled AI storage
17. Batched perception
18. Population profiling
19. NeuralModel contract
20. CPU neural runtime
21. Hybrid cognition
22. Neural batching
23. Advanced navigation
24. Group/social AI
25. GPU acceleration only where profiling justifies it

Do not begin with neural processing.

The entire ordinary AI pipeline should exist and be measurable first.

---

46. Implementation Dependencies

The major dependency chain is:

Entity Runtime
      |
      +-------------------+
      |                   |
      v                   v
World Interface        Controller
      |                   |
      v                   v
Perception            Physics
      |
      v
Memory
      |
      v
Cognition
      |
      v
Goals
      |
      v
Navigation
      |
      v
MotorIntent
      |
      +-------> Controller

Scheduler, LOD, batching, and LuaJIT integration operate around this pipeline rather than replacing it.

---

47. Final Architectural Contract

The AI architecture is governed by the following rules.

Rule 1 - World state is authoritative

World -> AIWorldInterface -> AI

AI does not derive simulation truth from rendering state.

Rule 2 - Perception filters the world

World Query -> Perception -> SensoryState

Cognition receives useful observations rather than raw engine state.

Rule 3 - Cognition is replaceable

SensoryState + Memory
          |
          v
   CognitiveModel
          |
          v
   CognitiveOutput

Rule 4 - Cognition selects intent, not physical state

Cognition -> Goal -> Navigation -> MotorIntent

Rule 5 - Physics executes physical consequences

MotorIntent -> Controller -> Physics -> World

Rule 6 - AI and VGP remain separate

World
 |
 +-- VGP -> Geometry / Rendering
 |
 +-- AI  -> Entity Intelligence

Rule 7 - LuaJIT extends; C++ executes

LuaJIT
  |
  v
Configuration / Behavior Definition
  |
  v
C++ AI Runtime

Rule 8 - Expensive cognition is optional

Simple entities must not pay for advanced cognition they do not use.

Rule 9 - Scale through scheduling and LOD first

Before optimizing individual algorithms:

Do less work
    |
    v
Do work less often
    |
    v
Batch necessary work
    |
    v
Optimize measured bottlenecks
    |
    v
Move appropriate workloads to GPU

Rule 10 - Preserve the abstraction seam

Future AI approaches should plug into:

CognitiveModel

rather than requiring the entity architecture to be redesigned.

---

48. V1-V4 Summary

V1 - FUNCTIONAL AI
------------------
Perception
Memory
State machines
Utility AI
Goals
Navigation interface
MotorIntent
Controller integration
Lua configuration

             |
             v

V2 - SCALABLE AI
-----------------
Scheduler
AI LOD
Events
Pooling
Batching
Population simulation
Profiling

             |
             v

V3 - ADVANCED AI
-----------------
Optional neural cognition
Hybrid cognition
Neural batching
Advanced simulation

             |
             v

V4 - WORLD-SCALE AI
--------------------
Group/social systems
Advanced perception
Hierarchical navigation
Large populations
Adaptive behavior
Optional GPU acceleration

The resulting system takes the useful architectural lesson from the neural simulation work we examined — separate sensory stimulation, cognition, behavioral interpretation, and physical execution — without making that project, its biological model, or any particular neural representation part of the engine.

The result is a native C++ entity-intelligence framework that can begin with deterministic conventional AI, scale through scheduling and LOD, selectively adopt neural processing where it provides measurable value, and accept entirely different cognition systems later through the same "CognitiveModel" boundary.