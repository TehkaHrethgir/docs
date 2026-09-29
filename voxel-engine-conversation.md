# Voxel Engine Design & Build Conversation

---

## C++ + LuaJIT — build/practices advice

### Architecture / data layout
- **Chunk storage**: flat arrays (`uint16_t[]`/`uint8_t[]`), palette-compressed per chunk. Biggest single performance lever.
- **Meshing**: greedy meshing (binary greedy meshing is a fast bitmask-based variant), run on background thread pool per dirty chunk, never on main thread.
- **Threading model**: world-gen, meshing, and Lua mod ticks are three concurrency domains. Common pattern: main thread (render/input) → job system (gen + meshing, no Lua) → single Lua thread fed via a queue from workers (Lua states aren't thread-safe).

### LuaJIT-specific practices
- Use FFI for hot-path *data*, keep the mod-facing API higher-level (functions/tables) so you don't leak internal memory layout into the modding contract.
- Sandbox aggressively: strip `os`, `io`, `require`; give each mod its own `lua_State` or sandboxed `_ENV`.
- Watch for GC pauses under load from per-tick allocation in mod scripts — document "avoid per-tick allocation" for modders.
- Version-pin LuaJIT, vendor it into the build (don't rely on system packages) given its slower release cadence.

### Build system
- **CMake** as the pragmatic default.
- **vcpkg or Conan** for dependency management.
- LuaJIT as a submodule/FetchContent pinned to a specific commit.

### Mod API design
- Version your mod API from day one (`api_version`).
- Design around events/hooks (`on_block_place`, `on_tick`, `on_chunk_load`) rather than polling.
- Split data-driven (JSON/TOML: blocks, items, recipes, loot tables) vs. code-driven (Lua: behavior) content early.

---

## Full folder layout, palette compression, Lua sandboxing, Lua+JSON hybrid

**Q:** sketch an actual folder/module layout, then go deeper on each of these: chunk palette compression scheme, Lua sandboxing pattern. Also can I have both Lua and JSON? Lua for complex mods, JSON for simple items/structures using 2D blueprints stacked to create 3D structures.

**A:**

### Folder / Module Layout

```
voxel-engine/
├── CMakeLists.txt
├── vcpkg.json                    # or conanfile.txt
├── engine/
│   ├── core/
│   │   ├── job_system.{h,cpp}    # thread pool for gen/meshing
│   │   ├── event_bus.{h,cpp}     # engine-wide event dispatch
│   │   └── logger.{h,cpp}
│   ├── world/
│   │   ├── chunk.{h,cpp}         # chunk data + palette
│   │   ├── chunk_manager.{h,cpp} # load/unload/streaming
│   │   ├── world_gen.{h,cpp}
│   │   └── block_registry.{h,cpp}
│   ├── render/
│   │   ├── mesher.{h,cpp}        # greedy meshing
│   │   ├── renderer.{h,cpp}
│   │   └── chunk_mesh_cache.{h,cpp}
│   ├── physics/
│   │   └── voxel_collision.{h,cpp}
│   ├── scripting/
│   │   ├── lua_vm.{h,cpp}        # lua_State lifecycle, sandbox setup
│   │   ├── lua_bindings/         # one file per exposed API surface
│   │   │   ├── bind_world.{h,cpp}
│   │   │   ├── bind_entity.{h,cpp}
│   │   │   └── bind_events.{h,cpp}
│   │   └── mod_loader.{h,cpp}    # discovers mods, resolves load order
│   └── data/
│       ├── json_loader.{h,cpp}   # generic schema-validated JSON loader
│       ├── blueprint.{h,cpp}     # 2D-layer-stack structure format
│       └── registry.{h,cpp}      # generic ID<->definition registry
├── mods/
│   ├── core/                     # your own "base game" as a mod
│   │   ├── mod.json               # metadata + api_version
│   │   ├── items/*.json
│   │   ├── blueprints/*.json
│   │   └── scripts/*.lua
│   └── example_mod/
├── assets/
└── tests/
    ├── unit/
    └── mod_compat/                # run every bundled mod against current API
```

Key principle: your own base content ships as a mod, using the same pipeline everyone else uses — forces the mod API to actually be sufficient.

### Chunk Palette Compression

```cpp
struct ChunkPalette {
    std::vector<BlockID> palette;      // small list of distinct block IDs used
    std::vector<uint8_t> indices;      // per-voxel index into palette (bit-packed)
    uint8_t bits_per_index;            // computed from palette.size()

    BlockID get(int x, int y, int z) const {
        size_t i = flatten(x, y, z);
        uint32_t idx = read_packed(indices, i, bits_per_index);
        return palette[idx];
    }

    void set(int x, int y, int z, BlockID id) {
        auto it = std::find(palette.begin(), palette.end(), id);
        if (it == palette.end()) {
            palette.push_back(id);
            if (palette.size() > (1u << bits_per_index)) grow_bits(); // repack
            it = palette.end() - 1;
        }
        write_packed(indices, flatten(x,y,z), bits_per_index, it - palette.begin());
    }
};
```

A chunk with only 4 distinct blocks needs 2 bits/voxel instead of 16 — an 8x reduction on a 32³ chunk (8KB vs 64KB); all-air/all-stone chunks compress to almost nothing.

Practical notes:
- Start `bits_per_index` at 0, grow (0→1→2→4→8→16) only as distinct block count demands.
- Cap the palette (e.g. 256 entries), fall back to a direct array past that.
- Keep a global block registry mapping `BlockID -> definition` so the palette stores small IDs only.
- Same scheme Minecraft itself uses (palette + bit-packed long array).

### Lua Sandboxing Pattern

```cpp
struct ModContext {
    lua_State* L;
    std::string mod_id;
};

ModContext create_mod_sandbox(lua_State* global_L, const std::string& mod_id) {
    lua_State* L = lua_newthread(global_L);   // coroutine-style isolated stack,
                                                // shares the global VM's memory/GC

    lua_newtable(L);
    int sandbox_env = lua_gettop(L);

    const char* safe_globals[] = {
        "pairs", "ipairs", "type", "tostring", "tonumber",
        "table", "string", "math", nullptr
    };
    for (int i = 0; safe_globals[i]; i++) {
        lua_getglobal(global_L, safe_globals[i]);
        lua_xmove(global_L, L, 1);
        lua_setfield(L, sandbox_env, safe_globals[i]);
    }
    // Deliberately omit: io, os, package, require, load, dofile, debug

    lua_pushcfunction(L, mod_require);        // resolves only within this mod's folder
    lua_setfield(L, sandbox_env, "require");

    register_engine_api(L, sandbox_env, mod_id);

    lua_setfenv(L, -2);  // Lua 5.1/LuaJIT idiom for per-thread environment

    return { L, mod_id };
}
```

Key decisions:
- One `lua_State` (coroutine) per mod, sharing one parent VM/GC — cheaper than fully separate VMs, gives per-mod isolation via distinct `_ENV`. Hard isolation against a determined attacker needs separate OS processes + IPC.
- Explicit whitelist, not blacklist.
- Custom `require` resolves only inside the mod's own directory tree.
- Resource limits: `lua_sethook` with instruction count to kill runaway loops; custom allocator per mod for a memory ceiling.

### Lua + JSON Hybrid

Same pattern as Minecraft (Forge/Fabric code mods + vanilla datapacks).
- **JSON**: declarative static data — items, block properties, recipes, loot tables, structure blueprints.
- **Lua**: anything with behavior — tick logic, AI, event hooks, world-gen mods, UI.

Blueprint format (2D layers stacked into 3D):

```json
{
  "id": "core:watchtower",
  "size": { "x": 5, "y": 8, "z": 5 },
  "palette": {
    "#": "core:stone_brick",
    ".": "core:air",
    "W": "core:wood_planks",
    "T": "core:torch"
  },
  "layers": [
    ["#####", "#####", "#####", "#####", "#####"],
    ["#...#", "#...#", "#...#", "#...#", "#####"],
    ["#...#", "#..T#", "#...#", "#...#", "#####"]
  ]
}
```

Loader: for each Y layer → each row (Z) → each character (X), look up palette, call `world.set_block`. JSON-only modders can build structures with zero Lua; Lua mods can call `place_blueprint(id, position, rotation)` programmatically.

Refinements: support rotation/mirroring at placement time; allow blueprints to reference sub-blueprints; optional `"entities"` array for spawn points/loot; version the blueprint schema (`"schema_version"`).

---

## 6. World-gen hooks, chunk-boundary structures, mod.json format

**Q:** "the world-gen mod hooks" / "how blueprint placement should interact with chunk boundaries" / "the mod.json metadata/load-order format"

**A:**

### World-Gen Mod Hooks

Staged pipeline, not a monolithic function:

```cpp
enum class GenStage {
    Terrain, Carve, Surface, Decoration, Structure, Post
};
```

Mods register callbacks per stage with a priority:

```lua
world.register_gen_hook("decoration", function(chunk, ctx)
    for x, z in chunk:columns() do
        if ctx.biome(x, z) == "core:desert" and ctx.rng:chance(0.02) then
            world.place_blueprint("mymod:cactus_cluster", {x=x, y=ctx.surface_height(x,z), z=z})
        end
    end
end, { priority = 100 })
```

**Determinism** is a hard requirement: derive every mod's RNG from `hash(world_seed, chunk_coord, mod_id, purpose)`, never wall-clock time; don't let mods query neighboring ungenerated chunks during early stages.

**Async-friendly hook contract**: since gen runs on worker threads, and Lua is typically kept single-threaded, use a two-phase approach — raw terrain math in pure C++ on workers, then queue "decorate this chunk" jobs drained on the single Lua thread. (Alternative: a Lua state per worker thread, more complex, only worth it if profiling shows a real bottleneck.)

### Structures Spanning Chunk Boundaries

Deferred placement via a pending-write buffer:

```cpp
struct PendingStructureWrite {
    ChunkCoord target_chunk;
    std::vector<BlockWrite> writes;
};

void place_blueprint(const Blueprint& bp, WorldPos origin) {
    std::unordered_map<ChunkCoord, std::vector<BlockWrite>> by_chunk;
    for (auto& [local_pos, block_id] : bp.iterate_blocks()) {
        WorldPos world_pos = origin + local_pos;
        ChunkCoord cc = chunk_coord_of(world_pos);
        by_chunk[cc].push_back({ local_within_chunk(world_pos), block_id });
    }
    for (auto& [cc, writes] : by_chunk) {
        if (auto* chunk = chunk_manager.get_if_loaded(cc)) {
            apply_writes(*chunk, writes);
        } else {
            pending_writes[cc].push_back({ cc, writes });
        }
    }
}

Chunk* ChunkManager::load_or_generate(ChunkCoord cc) {
    Chunk* chunk = generate_chunk(cc);
    if (auto it = pending_writes.find(cc); it != pending_writes.end()) {
        for (auto& pw : it->second) apply_writes(*chunk, pw.writes);
        pending_writes.erase(it);
    }
    return chunk;
}
```

This is the same pattern Minecraft uses for cross-chunk structure pieces.

Refinements: decide full bounding box before writing anything (Structure stage, finalized terrain data); reserve the footprint first via a spatial index so mods' structures don't overlap; persist the pending-write queue to disk (not just in-memory) so half-built structures survive a restart; force-generate synchronously for intentionally huge blueprints.

### `mod.json` Metadata & Load Order

```json
{
  "id": "core",
  "name": "Core Game Content",
  "version": "1.4.2",
  "api_version": "^2.0",
  "authors": ["you"],
  "description": "Base game blocks, items, and structures.",

  "dependencies": {
    "engine_core": ">=2.0.0"
  },
  "optional_dependencies": {
    "some_lib_mod": ">=1.0.0"
  },
  "incompatible_with": ["legacy_core"],

  "load_order": {
    "before": [],
    "after": ["engine_core"]
  },

  "entrypoints": {
    "scripts": ["scripts/main.lua", "scripts/worldgen.lua"],
    "data": {
      "items": "items/",
      "blueprints": "blueprints/"
    }
  }
}
```

- `api_version` uses a semver range (`^2.0`), not an exact pin — avoids breaking every mod on every engine patch release.
- `dependencies` (hard, fails to load if missing) vs. `optional_dependencies` (soft, detect-and-integrate).
- `load_order.before`/`after` for pure ordering without forcing a hard dependency.
- `incompatible_with` for explicit conflict declaration.

Resolving load order via topological sort:

```cpp
std::vector<ModID> resolve_load_order(const std::vector<ModManifest>& mods) {
    Graph<ModID> dep_graph;
    for (auto& mod : mods) {
        dep_graph.add_node(mod.id);
        for (auto& dep : mod.dependencies) dep_graph.add_edge(dep, mod.id);
        for (auto& after : mod.load_order.after) dep_graph.add_edge(after, mod.id);
        for (auto& before : mod.load_order.before) dep_graph.add_edge(mod.id, before);
    }
    auto order = topological_sort(dep_graph);
    if (!order) throw ModLoadError("Circular dependency: " + order.cycle_description());
    return *order;
}
```

Detect cycles and fail loudly with actual mod names; validate `api_version` compatibility before running any mod code; two-phase load per mod (register phase for all mods' JSON/IDs, then init phase runs Lua `main.lua`) so cross-mod references are load-order-independent.

---

## 7. Entity system, save/serialization, multiplayer sync

**Q:** Continue

**A:**

### Entity System + Lua API

ECS as the foundation (recommend `entt` — mature, header-only):

```cpp
using EntityID = uint32_t;

struct TransformComponent { Vec3 pos, velocity; float yaw, pitch; };
struct HealthComponent { float current, max; };
struct LuaBehaviorComponent { std::string mod_id, script_ref; int lua_ref; };

class World {
    entt::registry registry;
public:
    EntityID spawn_entity(const std::string& type_id, Vec3 pos);
    template<typename T> T& get_component(EntityID);
    template<typename T> bool has_component(EntityID);
};
```

Lua-facing:

```lua
entity.register_type("mymod:zombie", {
    base_health = 20,
    on_spawn = function(e) e:set_ai_goal("wander") end,
    on_tick = function(e, dt)
        local player = e:nearest_player(16)
        if player then e:set_ai_goal("chase", player) end
    end,
    on_damage = function(e, amount, source) return amount end
})
```

Per-frame `on_tick` for every entity is a real perf risk at scale — mitigate with staggered ticks, distance-based tick rate, and keeping hot-path physics in C++ (only AI decision-making goes through Lua).

### Save / Serialization & Mod Data Versioning

Save the palette representation directly:

```
[chunk header: coord, version, timestamp]
[palette: count, then N block-id strings or interned IDs]
[bits_per_index]
[packed indices blob]
[entity list: serialized ECS components]
[block entities / tile data]
```

**Never save raw numeric block IDs** — save string identifiers (`"core:stone"`) in a per-save ID table, map string↔numeric at load time. Missing mod IDs on load should degrade gracefully (placeholder), not crash.

Mod data schema versioning with migrations:

```json
{ "save_schema_version": 3 }
```

```lua
mod.register_migration(2, 3, function(old_data)
    old_data.armor_value = old_data.defense * 2
    return old_data
end)
```

Same pattern as database migrations, applied per-mod.

### Multiplayer Sync Considerations

- Server-authoritative simulation always — Lua ticks, world-gen, physics run only on server.
- Delta sync per chunk/entity, not full chunk resend.
- Mods must be sent to/verified against clients before join (server advertises mod list + versions).
- Reinforces the determinism requirement — any nondeterminism causes server/client desync.

---

## 8. entt-based component registration API + vertical slice plan

**Q:** Entt-based component registration API, then we will sketch a vertical slice.

**A:**

### entt-Based Component Registration API

Bridge layer: one entt component type (`LuaComponent`) holding a Lua table, keyed by mod-defined component name, since entt components are normally compile-time types but mods need runtime-defined ones.

```cpp
struct LuaComponent {
    std::string component_type;
    int lua_table_ref;
};

struct TransformComponent { Vec3 pos, velocity; };  // native, stays a real entt component
struct HealthComponent { float current, max; };

struct ComponentSchema {
    std::string name;
    std::vector<FieldDescriptor> fields;
    std::string owning_mod_id;
};
```

Lua-facing registration:

```lua
component.register("mymod:armor", {
    fields = {
        defense    = { type = "number", default = 0 },
        durability = { type = "number", default = 100 },
    }
})

entity.register_type("mymod:knight", {
    components = { "core:health", "mymod:armor" },
    on_spawn = function(e) e:set("mymod:armor", { defense = 5, durability = 100 }) end,
    on_damage = function(e, amount, source)
        local armor = e:get("mymod:armor")
        local reduced = math.max(0, amount - armor.defense)
        armor.durability = armor.durability - 1
        if armor.durability <= 0 then e:remove("mymod:armor") end
        return reduced
    end
})
```

C++ side uses entt's runtime `id_type`-keyed storage pools (`registry.storage<LuaComponent>(id)`) to get effectively unlimited distinct mod-defined component "slots" of the same underlying C++ type, without needing a compile-time type per mod component:

```cpp
class ComponentRegistry {
public:
    ComponentTypeID register_component(const std::string& name,
                                        std::vector<FieldDescriptor> fields,
                                        const std::string& mod_id) {
        ComponentTypeID id = next_id++;
        schemas[id] = { name, std::move(fields), mod_id };
        name_to_id[name] = id;
        return id;
    }

    void set_component(entt::registry& reg, EntityID e, const std::string& name, LuaTableRef data) {
        auto id = name_to_id.at(name);
        auto& pool = reg.storage<LuaComponent>(entt::id_type(id));
        pool.emplace(e, LuaComponent{ name, data.ref });
    }

    bool has_component(entt::registry& reg, EntityID e, const std::string& name) {
        auto id = name_to_id.at(name);
        return reg.storage<LuaComponent>(entt::id_type(id)).contains(e);
    }

private:
    std::unordered_map<ComponentTypeID, ComponentSchema> schemas;
    std::unordered_map<std::string, ComponentTypeID> name_to_id;
    ComponentTypeID next_id = 0;
};
```

Keep native performance-critical components as real compile-time entt types with direct C++ systems; reserve the `LuaComponent` runtime-slot mechanism for mod-added data read/written mostly from Lua hooks.

Query API — iterate smallest pool, filter by `contains()` on the rest (slower than entt's native static `view<A,B,C>()`, acceptable for occasional Lua-driven queries, not for hot-path native systems).

### Vertical Slice: Minimum Path

Scope: one hardcoded chunk (flat 16³ stone slab), naive per-cube mesh (no greedy meshing yet), one Lua-scripted block (`on_place` hook, logs a message). No multiplayer, no saving, no ECS entities yet.

Build order:
1. Window + render loop, hardcoded triangle.
2. Static chunk data structure, hardcoded with a hole punched in it.
3. Naive mesher (face culling, no greedy merge).
4. Camera + input (WASD + mouse look).
5. LuaJIT boot — trivial `print("hello from lua")`.
6. Bind `world.get_block(x,y,z)` / `world.set_block(x,y,z,id)`.
7. `block.register("test:glowstone", { on_place = ... })` wired to a raycast-based placement.
8. Confirm the full loop: place block → mesh updates → Lua hook fires and prints.

Everything else (palette compression, greedy meshing, job system, ECS, sandboxing, blueprints, multiplayer) becomes incremental work layered onto this proven skeleton.

---

## 9. Graphics API choice + concrete phase plan

**Q:** Let's discuss graphics API, and concrete the first phase of development, making sure to integrate services or plan to integrate further along in the build process. We will wrap back around to the CMake setup and initial `lua_vm.cpp` after we iron out more design choices.

**A:**

### Graphics API Choice

**OpenGL** — simplest to start, huge reference material, but aging, awkward multithreaded rendering, deprecated on macOS, no path to modern GPU features.

**Vulkan** — explicit control, multithreaded command recording, best performance ceiling, cross-platform (MoltenVK on macOS) — but brutal boilerplate (800–1500 lines for "hello triangle"), steep learning curve, easy sync bugs.

**A wrapper (bgfx, Diligent Engine, wgpu-native/Dawn, SDL_GPU)** — modern explicit rendering model, abstracts platform backend selection, cuts boilerplate:
- **bgfx**: mature, shipped in real games, backend-agnostic (D3D11/12, Vulkan, Metal, OpenGL, consoles), own shader language/toolchain.
- **Diligent Engine**: closer to raw Vulkan/D3D12 semantics, supports HLSL directly across backends.
- **wgpu (Dawn/wgpu-native)**: implements the WebGPU spec, modern explicit API, WGSL, includes a free path to browser deployment.

**Recommendation**: start with a wrapper (bgfx or wgpu-native), not raw Vulkan — engineering time is better spent on world/scripting/meshing systems than re-deriving Vulkan boilerplate; a wrapper still gives the multithreaded command-recording model needed; migrating to raw Vulkan later (if profiling shows a real CPU-side draw-call submission ceiling) is a well-trodden path, whereas starting raw Vulkan and wishing you'd shipped faster has no equivalent escape hatch. Between the two: **bgfx** for maturity/shipped-game track record (unless a web target matters, which would favor wgpu).

### Concrete Phase Plan (with integration seams marked)

**Phase 0 — Project skeleton**: CMake + vcpkg/Conan, folder layout, window via GLFW/SDL. *Seam*: put windowing/input behind a thin interface (`IWindow`, `IInput`) for a future headless server build.

**Phase 1 — Renderer bring-up**: bgfx/wgpu integration, hardcoded triangle → textured cube. *Seam*: design vertex/buffer upload as chunk-mesh-shaped (`ChunkMesh { vertex_buffer, index_buffer, draw_call }`) from the start.

**Phase 2 — Static chunk + naive mesher**: hardcoded chunk data, face-culling-only mesher. *Seam*: `Chunk::get_block`/`set_block` as the only access path to block data, so palette compression later is localized.

**Phase 3 — Camera + input, basic raycasting**: WASD/mouse-look, raycast for block picking.

**Phase 4 — LuaJIT boot + minimal bindings**: embed LuaJIT, `world.get_block`/`set_block`, `block.register(...)`/`on_place` hook. *Seam (the big one)*: build the sandboxing/mod-loader skeleton (per-mod `lua_State`, whitelist globals, `mod.json` parsing) from the start, even with only one test mod — retrofitting sandboxing after real mods exist is a breaking change.

**Phase 5 — Job system + threading model**: thread pool, move meshing off main thread. *Seam*: decide main-thread-only (Lua, GPU resource creation) vs. safely-parallel (terrain math, meshing) now; commit to "Lua single-threaded, workers feed via queue" via an explicit `MainThreadQueue` type.

**Phase 6 — Palette compression + greedy meshing**: swap in real implementations behind the Phase 2/5 seams without touching calling code.

**Phase 7 — World-gen pipeline + staged hooks**: real terrain gen, staged pipeline with mod hooks. *Seam*: build the pending-structure-write queue even before real blueprints exist, tested with a trivial 2-chunk-spanning hardcoded structure.

**Phase 8 — JSON data loader + blueprints**: item/block JSON registration, blueprint format, `mod.json` parsing, load-order resolution.

**Phase 9 — ECS + ComponentRegistry**: entt integration, native components, the runtime `LuaComponent` slot mechanism. Deliberately late — benefits from real mod-authoring experience from Phases 4–8.

**Later/deferred**: multiplayer (server-authoritative split, delta sync — but determinism discipline should be enforced from Phase 7 onward even in singleplayer); save/serialization + mod data migrations (straightforward once the Phase 8 ID-table discipline is in place).

---

## 10. setup walkthrough (hand-typed, teaching mode)

**Q:** I will be using VGP. Walk me through the process — I will be typing the code out by hand for comprehension. You're the teacher, let's build this together.

**A:**

### Lesson 1: Tooling & Project Skeleton

**Prerequisites**: C++17+ compiler (MSVC/GCC/Clang), CMake 3.20+, Git. GLFW pulled in via CMake, not manually installed.

**Why bgfx needs a different build approach**: bgfx's canonical build system is GENie/premake-based, not CMake natively. We use the community-maintained CMake wrapper `bkaradzic/bgfx.cmake` (wraps bgfx+bimg+bx) rather than hand-rolling GENie integration.

**Step 1 — Directory structure**:
```
voxel-engine/
├── CMakeLists.txt
├── src/
│   └── main.cpp
└── third_party/
```
CMake's `FetchContent` pulls bgfx.cmake and GLFW into `third_party` automatically.

**Step 2 — Root `CMakeLists.txt`**:

```cmake
cmake_minimum_required(VERSION 3.20)
project(VoxelEngine LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)

FetchContent_Declare(
    glfw
    GIT_REPOSITORY https://github.com/glfw/glfw.git
    GIT_TAG 3.4
)
set(GLFW_BUILD_DOCS OFF CACHE BOOL "" FORCE)
set(GLFW_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(GLFW_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(glfw)

FetchContent_Declare(
    bgfx.cmake
    GIT_REPOSITORY https://github.com/bkaradzic/bgfx.cmake.git
    GIT_TAG master
)
FetchContent_MakeAvailable(bgfx.cmake)

add_executable(voxel_engine src/main.cpp)

target_link_libraries(voxel_engine PRIVATE glfw bgfx bx bimg)
```

**Reasoning per line**:
- `CMAKE_CXX_STANDARD 17` — bgfx and modern voxel-engine idioms want at least C++17.
- `FetchContent_Declare`/`FetchContent_MakeAvailable` — CMake clones the repo at the given tag at configure time, adds it as if a local subdirectory.
- GLFW pinned to tag `3.4` (stable numbered releases); bgfx.cmake left on `master` for now since bgfx doesn't do traditional numbered releases — will pin to a specific commit hash once the build is confirmed working, for reproducibility.
- The three `GLFW_BUILD_... OFF` lines disable GLFW's tests/docs/examples, set *before* `FetchContent_MakeAvailable` since that's when GLFW's own `CMakeLists.txt` runs.
- `bx` (base/utility) and `bimg` (image loading) are bgfx's two companion libraries, always required alongside `bgfx` — bgfx.cmake builds all three as separate targets.

**Step 3 — placeholder `main.cpp`**:

```cpp
#include <cstdio>

int main() {
    printf("Voxel engine skeleton compiles.\n");
    return 0;
}
```

**Checkpoint**: type both files, then:

```
mkdir build
cd build
cmake ..
cmake --build .
```

The first `cmake ..` will take a while (cloning GLFW + bgfx/bx/bimg source trees) — expected, not an error. Report back whether *configure* succeeded and separately whether *build* succeeded (bgfx.cmake is the piece most likely to throw platform-specific issues) before moving on to window creation.

---

