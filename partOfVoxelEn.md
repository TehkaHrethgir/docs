Voxel Engine: implementation specification and V1–V4 roadmap

This is the handoff plan for an implementation agent. It incorporates the decisions made throughout this conversation, including VGP, meshers, JSON-defined buildings, LuaJIT modding, saves, entities, and the chosen Lua security tradeoff.

Implementation status is unknown. The agent must inspect the repository first and mark each item implemented, partial, or missing. This document defines the target behavior; it does not imply that a system is already built.


---

1. Fixed decisions

1. Build the engine in C++ from the ground up. Use the graphics API already selected or present in the repository. Do not add bgfx.


2. Use VGP (Voxel Geometry Pipeline) for rendering. VGP owns persistent GPU geometry pages, allocation, upload, visibility, LOD selection, and indirect submission. WorldState owns voxels; meshers produce geometry for VGP.


3. Embed LuaJIT in the engine process for Lua mods, with FFI available. Lua mods are off by default and require an explicit risk acknowledgment in the mod menu. They are not OS-sandboxed.


4. Support JSON-only mods for definitions such as blocks, items, recipes, loot, and buildings. JSON-only mods do not require enabling Lua.


5. Use stable namespace:path content IDs in definitions and saves. Runtime numeric IDs are temporary.


6. Keep generation, meshing, physics, and frequent entity updates in C++. Run Lua hooks at deliberate event or decision points.


7. Do not make a connectome or neural model a prerequisite for entity AI. Preserve a replaceable decision interface for later experiments.



Consequence of the Lua decision

Enabled FFI can call native functions with the game process’s permissions. The mod menu must state that clearly. An engine API can validate normal mod actions and prevent mistakes, but cannot enforce an OS boundary against a malicious FFI mod. LuaJIT’s documentation identifies FFI as unsafe for sandboxing. 


---

2. Architecture contracts

Module	Owns	Interface to the next module

ContentRegistry	Stable IDs, definitions, runtime IDs, missing-content mapping	Validated block/item/entity definitions
WorldState	Authoritative chunks, block values, edit revisions	Read-only WorldView; validated mutation operations
WorldGeneration	Seeded terrain, features, structure plans	Proposed writes, committed through WorldState
Meshers	Visible surface generation	Renderer-independent MeshData
VGP	GPU pages, uploads, allocation retirement, draw metadata, culling, indirect draws	Visible geometry; never authoritative block state
Simulation	Fixed ticks, entities, collision, AI schedule	Events and validated requests
ModRuntime	Manifest loading, JSON registration, Lua VM and callbacks	Definitions, event handlers, action requests
Persistence	Versioned world/chunk/entity records, migrations	Stable-ID translation and restored state
Networking, later	Authoritative replication and interest management	Validated actions and deltas


A public API call such as world.set_block must route through WorldState’s mutation path. It cannot write directly into chunk storage or VGP metadata.

Block change
  → validate action and block ID
  → update WorldState
  → increment chunk revision
  → invalidate affected chunk and neighbors at boundaries
  → schedule meshing from a revision-tagged read view
  → discard stale result if the world changed
  → allocate/upload geometry in VGP
  → replace draw metadata
  → retire previous allocation after GPU use completes


---

3. Content identity, registration, and saves

Stable IDs

Persistent definitions use namespace:path, for example core:stone and watchtowers:oak_tower. A mod may register IDs only in its own namespace. A content kind is part of the identity: a block and an item may share the same text ID without being the same definition.

Reserve core:air.

Give unknown saved blocks a distinct missing-block runtime representation.

Assign compact runtime IDs when registration finishes.

Never write runtime IDs to disk as permanent identities.

Keep the original stable ID for missing content so reinstalling a mod can restore it.

Handle ID renames through explicit migrations; do not silently repurpose old IDs.


Loading sequence

1. Read and validate mod manifests without executing Lua.


2. Resolve required dependencies, incompatible mods, API ranges, and deterministic load order.


3. Register JSON definitions and stable IDs for all selected mods.


4. Resolve references between definitions; report missing required references.


5. Finalize registries and assign runtime IDs.


6. Read the save’s content-ID table and create save-ID → runtime-ID maps.


7. If Lua mods have been explicitly enabled, initialize Lua and attach behavior callbacks.


8. Load or generate chunks and start simulation.



This permits JSON-only mods to load while Lua execution remains disabled.

Initial save format

Define fixed byte order, integer widths, coordinate encoding, record length limits, and format versions. Do not serialize raw C++ structs.

World header:
  magic, format version, world seed, world identity
  saved content-ID table: save-local number → stable kind and namespaced ID
  enabled generation-mod identity/version information

Chunk record:
  record version, chunk coordinate, revision
  palette of save-local block IDs
  voxel indices, integrity check
  later: block entities and structure state

Save entities and mod data in versioned records as they are added. A missing definition must retain enough original information for later restoration.


---

4. JSON-defined buildings: complete contract

A building is a blueprint: a palette of symbols plus 2D X/Z rows stacked into Y layers. JSON defines its shape. C++ validates and compiles it into a compact, immutable Blueprint before any world placement occurs.

Example: a small watch post

This example is 5 blocks wide (X), 3 layers high (Y), and 5 blocks deep (Z). Each layer has five Z rows; each row has five X symbols.

{
  "schema_version": 1,
  "id": "watchtowers:watch_post",
  "size": { "x": 5, "y": 3, "z": 5 },
  "origin": { "x": 2, "y": 0, "z": 2 },
  "palette": {
    "#": "core:stone_brick",
    "W": "core:oak_planks",
    ".": "core:air"
  },
  "layers": [
    [
      "#####",
      "#####",
      "#####",
      "#####",
      "#####"
    ],
    [
      "#...#",
      "#...#",
      "#...#",
      "#...#",
      "##.##"
    ],
    [
      "WWWWW",
      "WWWWW",
      "WWWWW",
      "WWWWW",
      "WWWWW"
    ]
  ],
  "placements": [
    {
      "entity": "core:loot_container",
      "at": { "x": 2, "y": 1, "z": 2 },
      "data": { "loot_table": "watchtowers:basic_supplies" }
    }
  ]
}

Symbol and coordinate rules

layers[y][z][x] selects a symbol. Layer zero is the bottom.

A palette symbol resolves to a registered block ID and writes that block.

"core:air" explicitly clears an existing block.

A reserved symbol, for example " ", means skip this position and leave the world unchanged. It must not appear in the palette.

origin is the local point mapped to the requested placement position. For the example, the center of the floor maps to the world placement position.

Rotation is around the vertical Y axis, in quarter turns. Mirroring is applied in local coordinates before rotation. Define the exact transform order once and test all combinations.

placements are typed, schema-validated objects. The engine creates them only after the blocks are committed; a Lua function embedded in JSON is not permitted.


Loader implementation

1. Parse JSON with file size and nesting limits.


2. Validate schema_version, ID ownership, dimensions, number of layers, number of rows, row width, symbols, and origin bounds.


3. Resolve every palette and placement reference against the registries.


4. Compile rows into local block operations: (local position, stable/runtime block ID or skip).


5. Compute transformed dimensions and occupied footprint for each permitted rotation/mirror combination.


6. Store the compiled result in a blueprint registry keyed by stable ID.


7. Report errors with mod ID, file path, layer, row, and column.



Start with single-character ASCII symbols and bounded blueprint sizes. A later schema version can introduce multi-character symbols, sub-blueprints, or procedural parameters if actual content requires them.

Placement API

struct BlueprintPlacement {
    BlueprintId id;
    WorldPos origin;
    QuarterTurn rotation;
    bool mirror_x;
    PlacementPolicy policy;
};

PlacementResult plan_blueprint(
    const BlueprintPlacement& request,
    const WorldView& world
);

CommitResult commit_blueprint(
    const PlacementResult& plan,
    WorldState& world
);

Planning computes the complete footprint and proposed operations without mutating the world. It checks bounds, terrain support, overlap rules, and placement policy. Committing applies the accepted operation through WorldState’s normal mutation path, grouped by target chunk.

Buildings that cross chunk boundaries

Do not place only the portion belonging to whichever chunk happens to generate first. For generated structures:

1. Divide the world into deterministic planning regions larger than a chunk.


2. Derive structure candidates and rotation from hash(world_seed, region, structure_id, purpose).


3. Compute each candidate’s full bounding box before placing any portion.


4. Resolve competing footprints in a stable order.


5. For each target chunk, derive the operations from accepted plans intersecting that chunk.


6. Apply operations at a defined generation stage. Persist any accepted plan or outstanding operation that cannot be reconstructed reliably from the seed and saved generation rules.


7. Define how later player edits take precedence so regeneration never overwrites them.



The result must be identical whether chunks A and B load in either order, simultaneously, or on different launches. Lua mods may call world.place_blueprint(id, position, rotation) for explicit placement, subject to the normal game-action validation.

JSON-only building content versus Lua

A JSON-only mod can register the example blueprint and a JSON generation rule such as “attempt this structure in specified biomes at a defined frequency.” Lua is optional for custom placement logic or interactive behavior. The generator reads both forms into the same planning and placement system.


---

5. World generation and mod hooks

Use explicit stages, with stage names and permitted operations documented:

1. Terrain: base density/heights and block materials.


2. Carve: caves and voids.


3. Surface: top materials and biome finishing.


4. Structures: reserved footprints and blueprint operations.


5. Decoration: vegetation and small features.


6. Post: final local adjustments and validation.



Generation must not depend on worker completion order. Derive RNG streams from the world seed, region/chunk coordinate, mod ID, and purpose. Give Lua hooks deterministic inputs and a bounded output contract. Worker threads run C++ generation and meshing; Lua callbacks execute on the thread that owns the Lua VM, with results returned as proposed operations.

Document when a hook may inspect neighboring data. A hook must not block waiting for an ungenerated neighbor that itself needs the current job.


---

6. LuaJIT and the mod menu

Mod package layout

mods/watchtowers/
  mod.json
  blocks/
    ...
  items/
    ...
  blueprints/
    watch_post.json
  generation/
    watch_post_rules.json
  scripts/
    main.lua

A manifest states mod ID, version, engine API range, required/optional dependencies, incompatibilities, and data and Lua entrypoints. Determine whether a mod contains executable Lua before running any of its code.

User choice and startup behavior

Lua mods are disabled by default.

JSON-only content can load independently.

The mod menu lists each Lua mod and version and shows a clear risk warning before enabling it.

Persist the user’s choice, but ask again if the set of enabled executable mods changes through installation, update, or dependency selection.

Provide a launch/recovery option that disables all Lua mods before their code runs.

Logs identify the mod whose initialization or callback failed.

Never present Lua environments, separate lua_State objects, or API validation as OS sandboxing while FFI is enabled.


Suggested warning:

> Enable Lua mods? Lua mods run inside the game and may use LuaJIT FFI to call native code. A malicious mod could read or change files, use the network, or run programs with your user account’s permissions. Enable only mods you trust.



Engine-facing mod API

Use versioned event hooks such as on_block_place, on_chunk_load, scheduled on_tick, on_entity_spawn, and on_damage. Normal scripts request operations through checked APIs. Give mods stable handles and namespaced IDs rather than internal pointers in the documented API.

Do not call a Lua on_tick for every entity every rendered frame. Schedule behavior by simulation tick and entity importance; perform frequent movement, collision, and physics in C++.


---

7. VGP and mesher integration

VGP does not remove meshing. It removes dependence on a separately managed GPU mesh object and draw submission for every chunk.

Common mesher output

struct MeshData {
    ChunkCoord chunk;
    ChunkRevision source_revision;
    MaterialRanges materials;
    std::vector<PackedVertex> vertices;
    std::vector<uint32_t> indices;
    Bounds bounds;
};

The packing format is chosen after defining required shader attributes. All meshers emit the same logical contract; VGP decides where to put the data.

Mesher	Placement	Purpose

Naive visible-face	V1, retained	Correctness reference and debugging
Greedy, including a bitmask-based variant if benchmarks support it	V2	Production block terrain
LOD mesher	V3	Reduced distant geometry
Surface and transition mesher	V3 only if smooth terrain exists	Smooth surfaces and seams across detail levels
Special-purpose terrain mesher	V4 as needed	Terrain that cannot use ordinary block faces


GPU allocation lifecycle

Implement pages with free-space tracking, growth/relocation, upload batching, and per-chunk allocation handles. A chunk edit produces new geometry; VGP can replace or relocate that allocation without rebuilding unrelated chunks. Keep draw metadata separate from authoritative chunk data. Account for frames in flight before reusing freed GPU memory.

V1 starts with indirect submission. V2 adds GPU frustum/distance culling and command generation. V3/V4 hierarchy, occlusion, material grouping, and adaptive detail are gated by profiling and CPU-reference correctness checks.


---

8. Entities, AI, persistence, and multiplayer

Use native components for transforms, collision, health, and frequently updated state. Mod-defined component schemas can add validated data without requiring a new compiled C++ component type for every mod. Saved component values must be engine-owned data, not merely references to live Lua tables.

The AI contract is:

World/Physics observations
  → perception
  → memory/decision
  → goal
  → navigation
  → movement intent
  → C++ controller and physics

Lua can select or change goals through scheduled callbacks. Future decision models may replace that decision step without changing world, physics, or VGP ownership.

For multiplayer, the server owns world generation, Lua-driven game rules, physics, and authoritative edits. Clients receive compatible content IDs and state deltas. Joining a server must never silently enable or execute a downloaded Lua mod; the local Lua opt-in and disclosure still apply.


---

9. Milestone roadmap

Version	Required deliverable	Exit gate

V1: editable saved slice	C++ build; authoritative flat chunks; stable IDs; naive mesher; camera/editing; persistent VGP geometry and indirect path; versioned chunk save/load; engine-authored LuaJIT event test	Edit/reload correctness, missing-ID preservation, stale-job rejection, GPU allocation-lifetime stress, baseline metrics
V2: generated moddable world	Bounded jobs; palette storage where beneficial; greedy meshing; staged generation; mod manifests and two-phase registry load; JSON blocks/items/buildings; deterministic cross-chunk placement; Lua opt-in and mod menu; GPU frustum/distance culling	JSON-only mod works with Lua off; changing Lua mod set requires confirmation; structures survive load-order/restart tests; greedy mesh and GPU culling match CPU references
V3: entities and detail	Native ECS/simulation; scheduled Lua AI; schema-based mod component data; entity and mod save migrations; measured hierarchy/LOD; smooth-surface transitions only if applicable	Entity data survives save/migration; simulation budget measured; LOD does not alter authoritative collision or block state
V4: scale and network	Profile-driven VGP improvements; special-purpose meshers as needed; authoritative multiplayer, interest management and deltas; long-running stress coverage	Measured optimization gains; validated network actions; mod compatibility and local Lua consent; bounded memory/queues and recoverable saves


10. Instructions for the implementing agent

1. Inventory first. Report which contracts already have working code and tests. Do not replace the graphics stack or working systems just because this specification describes them.


2. Implement in dependency order. Complete stable IDs, the authoritative edit path, and the first versioned save before expanding mod content or generation.


3. Treat every JSON format as a versioned public contract. Provide example files and precise loader errors.


4. Keep naive meshing and CPU visibility checks as references while optimizing VGP.


5. For each increment, report changed behavior, verification results, performance measurements where relevant, and remaining limitations.


6. Apply the chosen Lua policy exactly: in-process LuaJIT with FFI, default-off Lua mods, explicit disclosure, mod-set re-confirmation, and recovery mode. Do not claim it prevents a malicious enabled mod from reaching the OS.



First concrete work package: inspect the repository, then implement or verify V1 stable content identity, authoritative block mutation, and save/load. After that, build the JSON blueprint compiler and placement tests against that world contract; the generation rules can then use the same placement mechanism in V2.