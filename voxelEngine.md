I reviewed the full progression of the design, including the earlier rendering/VGP discussion and the decisions we subsequently tightened. There are a few architectural corrections I would make before freezing the roadmap for an implementation agent.

The biggest issue is not that the current roadmap is wrong; it is that several concepts have gradually accumulated responsibilities that should be separated more explicitly.

Sanity check and recommended changes

1. Keep VGP as the Voxel Geometry Pipeline, but distinguish the pipeline from its spatial structure. Right now “VGP” sometimes means the entire voxel-to-render pipeline and sometimes specifically means the occupancy hierarchy. That will confuse an implementation agent. I recommend naming the internal hierarchy VGP Spatial Hierarchy (VSH) for documentation purposes. VGP remains the overall coordinator; VSH is the derived hierarchical occupancy/spatial acceleration structure inside it.

2. Correct the mesher dependency. The previous wording WorldState → VGP → Mesher is useful conceptually but dangerous as a software dependency. The mesher needs voxel samples from WorldState, not copied voxel data from VGP. The cleaner relationship is:

WorldState
                         /    \
                        /      \
                       ▼        ▼
                     VGP      Mesher
                      │          ▲
                      │ request  │ voxel access
                      └──────────┤
                                 │
                              MeshData

VGP coordinates when, where, at what LOD, and with which mesher geometry should be generated. The selected mesher reads authoritative voxel information through a read-only WorldView/RegionView supplied with the request. This prevents VGP from accidentally becoming a second voxel database.

3. Separate CPU spatial hierarchy from GPU hierarchical culling. Earlier we correctly recommended that hierarchical GPU culling should only be added when profiling demonstrates that flat GPU culling has become insufficient. That does not conflict with building the VGP hierarchy early. The CPU VSH is foundational for dirty propagation, queries and spatial organization. GPU hierarchical culling is an optimization and should remain profiling-gated.

4. Preserve the persistent paged GPU geometry heap from the earlier plan. This was stronger than the generic “geometry allocator” wording in the later roadmap. V1 should deliberately establish persistent allocations rather than recreating geometry buffers every frame. The allocator needs allocation, free/reuse, growth, shrinkage, relocation, fragmentation accounting, and GPU-safe retirement.

5. Introduce revisions/generations from the beginning. Asynchronous meshing creates an easy race: region revision 12 starts meshing, revision 13 occurs, then the revision-12 mesh finishes and overwrites newer geometry. World regions, meshing requests and completed mesh results therefore need revision IDs. Stale results must be discarded.

6. Add explicit backpressure. Dirty edits can be generated faster than meshing or GPU uploads can consume them. Dirty queues, meshing queues and upload queues need bounded work budgets, coalescing and priorities. Otherwise destruction-heavy gameplay can produce runaway latency and memory growth.

7. Treat chunk/region boundaries as a first-class dependency. An edit at a region edge can alter the visible faces of the neighboring region. Dirty propagation therefore needs neighbor invalidation from V1 rather than being added as a later optimization.

8. Keep CPU reference implementations longer than the roadmap currently implies. CPU frustum culling, naive meshing, exact CPU traversal and other simple paths should remain permanently available in debug/test builds as correctness oracles for optimized GPU/production implementations.

9. Move Transvoxel slightly later than basic LOD. First prove discrete multi-LOD generation and selection. Then solve transition geometry. Otherwise an agent can end up debugging LOD generation, selection and crack fixing simultaneously.

10. Make performance gates quantitative. “Works” is insufficient for an engine like this. Each phase needs reproducible stress scenes and metrics: median and p99 CPU frame time, GPU frame time, meshing latency, edit-to-visible latency, upload bandwidth, allocation fragmentation, VSH update cost, visible-region count and memory consumption.

With those corrections, I would give the implementation agent the following roadmap.

NATIVE C++ VOXEL ENGINE

Implementation Roadmap and Agent Execution Specification

Technology

Core implementation: C++
Modding/extensibility: Embedded LuaJIT
Primary voxel geometry architecture: VGP — Voxel Geometry Pipeline
Development model: Working vertical slices progressing through V1 → V4

---

1. ARCHITECTURAL DEFINITIONS

1.1 WorldState

WorldState is the authoritative representation of the world.

It owns voxel/material state and authoritative world mutations.

Everything derived from voxel state must ultimately be regenerable from WorldState.

WorldState
    │
    ├── voxel/material state
    ├── revisions
    ├── read/query interface
    ├── modification interface
    └── WorldChange events

---

2. VGP — VOXEL GEOMETRY PIPELINE

VGP is the coordination layer responsible for turning WorldState changes and spatial demand into appropriate renderable voxel geometry.

It answers:

WHAT region requires geometry?
WHEN does it require regeneration?
WHAT resolution is required?
WHICH mesher should process it?
WHAT geometry representation currently exists?

It does not answer:

What is the authoritative voxel value?
How are triangles actually generated?
Where are vertices physically stored on the GPU?
How is the final frame shaded?

Those belong to WorldState, Meshing, Geometry and Renderer respectively.

---

3. VSH — VGP SPATIAL HIERARCHY

Within VGP, define the VGP Spatial Hierarchy (VSH).

This prevents the term VGP from being overloaded.

VGP
│
├── VSH
│   ├── occupancy
│   ├── hierarchy
│   ├── spatial summaries
│   └── traversal
│
├── dirty-region tracking
├── work scheduling
├── meshing coordination
├── LOD coordination
├── geometry association
└── spatial query interface

VSH is derived state.

Initially:

EMPTY
OCCUPIED
MIXED

Later it may contain additional conservative summaries when profiling demonstrates value.

---

4. CORRECT DEPENDENCY MODEL

The authoritative dependency is:

                         WorldState
                         /       \
                        /         \
                       ▼           ▼
                     VGP        WorldView
                      │             │
                      │             ▼
                      │          Mesher
                      │             │
                      │             ▼
                      │          MeshData
                      │             │
                      ▼             ▼
                Geometry Record → Allocator
                                      │
                                      ▼
                                     GPU

VGP issues a meshing request.

The mesher obtains authoritative voxel samples through a read-only WorldView/RegionView.

VGP must not duplicate the voxel payload merely to feed the mesher.

---

5. MEShing REQUEST CONTRACT

Conceptually:

MeshingRequest

region
world_revision
requested_lod
mesher_type
neighbor_requirements
output_requirements
priority

The exact C++ representation is not frozen by this roadmap.

The important requirement is that every request identifies the WorldState revision from which its output is generated.

---

6. REVISION SAFETY

Every mutable world region must have a revision/generation.

Example:

Region revision 41
        │
        ▼
MeshingRequest(41)
        │
        │ world changes
        ▼
Region revision 42
        │
        ▼
MeshingRequest(42)

If result 41 finishes afterward:

MeshResult(41)
       ↓
current revision = 42
       ↓
DISCARD STALE RESULT

Never allow asynchronous jobs to overwrite newer world geometry.

This rule applies to:

WorldState
VGP work
meshing
geometry uploads
streaming

where asynchronous completion can occur.

---

7. REGION MODEL

Establish a logical spatial region/chunk abstraction early.

A region should eventually associate:

RegionID
bounds
world revision
VSH state
dirty state
requested LOD
resident LOD
meshing state
geometry handle(s)
streaming state

Do not require all of these fields to exist in one giant structure.

Prefer handles and subsystem-owned tables where appropriate.

---

8. BOUNDARY INVALIDATION

Voxel geometry crosses logical region boundaries.

Therefore:

edit interior voxel
       ↓
dirty owning region

but:

edit boundary voxel
       ↓
dirty owning region
       +
dirty affected neighbor

This must exist in V1.

Otherwise geometry can become stale along chunk boundaries.

---

9. ASYNCHRONOUS PIPELINE

The long-term CPU pipeline is:

World Changes
     ↓
Dirty Region Queue
     ↓
VGP Work Determination
     ↓
Meshing Request Queue
     ↓
Worker Threads
     ↓
Completed Mesh Queue
     ↓
GPU Upload Queue
     ↓
Geometry Allocator
     ↓
GPU

Every stage must support bounded processing.

No stage may assume unlimited work per frame.

---

10. BACKPRESSURE

If modifications occur faster than downstream processing:

edits > meshing throughput

the engine must not generate an infinitely growing queue.

Use:

dirty-region coalescing
revision replacement
priority
per-frame budgets
queue limits
stale-job rejection

Multiple edits to the same region should normally collapse into the newest required revision.

---

11. GEOMETRY ARCHITECTURE

The geometry path is:

Mesher
   ↓
MeshData
   ↓
Geometry Manager
   ↓
Persistent GPU Geometry Allocator
   ↓
GPU Vertex/Index Pages
   ↓
Render Metadata

Geometry must not be rebuilt into global monolithic buffers every frame.

---

12. PERSISTENT PAGED GPU HEAP

Establish persistent GPU geometry allocation from V1.

The allocator eventually needs:

allocate
free
reuse
grow
shrink
relocate
retire
compact

Conceptually:

GPU Vertex Pages
┌─────────────────────────────────┐
│ AAAA │ BBBBBBB │ free │ CCC │  │
└─────────────────────────────────┘

GPU Index Pages
┌─────────────────────────────────┐
│ AAA │ BBBBB │ free │ CCCCC │   │
└─────────────────────────────────┘

A region owns logical geometry handles, not raw permanent pointers into these pages.

Relocation must therefore be possible without changing region identity.

---

13. GPU-SAFE RETIREMENT

Never reuse geometry memory still referenced by an in-flight frame.

old allocation
      ↓
replaced
      ↓
retirement queue
      ↓
GPU completion
      ↓
free/reuse

This must be part of the allocator architecture from the beginning.

---

14. MESH DATA

All meshers emit a common renderer-independent intermediate representation.

Conceptually:

MeshData

vertices
indices
surface/material ranges
bounds
source revision
source region
LOD
statistics

MeshData must not know where its eventual GPU allocation lives.

---

15. MESHER FAMILY

The planned mesher family is:

Naive Face Mesher
        ↓
Greedy Mesher
        ↓
LOD Mesher
        ↓
Surface Mesher
        ↓
LOD Transition / Transvoxel
        ↓
Specialized Terrain Meshers

This is an implementation progression, not an inheritance hierarchy.

All meshers should target the same MeshData contract where practical.

---

V1 — CORRECTNESS + PERSISTENT GEOMETRY

16. V1 OBJECTIVE

Produce the first complete engine capable of modifying a voxel world and showing the resulting geometry.

Target vertical slice:

Engine
 ↓
WorldState
 ↓
WorldChange
 ↓
VGP
 ↓
Meshing
 ↓
Persistent GPU Allocation
 ↓
Renderer
 ↓
Frame

LuaJIT must also be capable of making a world modification through the public API.

---

17. V1 FOUNDATION

Implement:

repository/build system
platform abstraction
logging/assertions
error/result handling
math
memory foundations
handles/IDs
filesystem
timing
profiling markers
unit-test infrastructure

Do not build elaborate custom replacements for standard C++ facilities without demonstrated need.

---

18. V1 PLATFORM LOOP

Establish:

startup
 ↓
window
 ↓
input
 ↓
update
 ↓
render
 ↓
present
 ↓
shutdown

Produce a working executable before introducing voxel systems.

---

19. V1 GPU FOUNDATION

Implement the minimum graphics abstraction required for:

device
swapchain
buffers
textures
shaders
pipelines
bindings
command recording
submission
synchronization
presentation

Then render a primitive.

This is the first vertical milestone.

---

20. V1 JOB SYSTEM

Implement a basic worker pool and job interface.

Initial requirements:

submit
worker execution
completion/fence
clean shutdown
profiling

Add sophisticated scheduling only when actual workloads require it.

---

21. V1 LUAJIT FOUNDATION

Embed LuaJIT and establish an explicit C++ API boundary.

Initial Lua capabilities:

mod discovery
script execution
event registration
logging
content registration
World API access

Lua must modify voxel state through the public World API.

Never expose raw VGP, allocator or GPU structures.

---

22. V1 WORLDSTATE

Implement authoritative voxel storage.

Minimum operations:

read
write
region read
region modification
revision increment
WorldChange emission

First optimize correctness and predictable access patterns.

---

23. V1 WORLDVIEW

Introduce a read-only view suitable for meshing.

Conceptually:

WorldView
    │
    ├── read voxel
    ├── read neighbor
    ├── query material
    └── identify source revision

This prevents meshers from mutating WorldState.

---

24. V1 VSH

Implement the first VGP Spatial Hierarchy.

Begin with:

L0 occupancy
 ↓
parent reduction
 ↓
higher hierarchy

Reduction:

all empty       → EMPTY
all occupied    → OCCUPIED
mixed children  → MIXED

Do not prematurely add rich material data.

---

25. V1 DIRTY PROPAGATION

WorldChange produces spatial invalidation:

WorldChange
     ↓
affected region(s)
     ↓
dirty VSH
     ↓
dirty geometry

Boundary edits must invalidate appropriate neighboring regions.

---

26. V1 INCREMENTAL VSH UPDATE

A localized modification must update only the necessary hierarchy path.

Critical test:

large world
+
one voxel edit
        ↓
small bounded update

A full hierarchy rebuild fails this requirement.

---

27. V1 NAIVE MESHER

Implement naive visible-face generation first.

Keep it permanently as the block-voxel correctness oracle.

Its job is not to be fast.

Its job is to be obviously correct.

---

28. V1 GREEDY MESHER

Once naive output is validated, implement greedy face merging.

Compare against naive output for surface equivalence.

Measure:

input voxels
visible faces
merged quads
vertices
indices
CPU time

---

29. V1 GEOMETRY HEAP

Implement persistent paged GPU allocation.

Minimum production capabilities:

allocation
free/reuse
upload
replacement
growth
shrink
GPU-safe retirement
fragmentation statistics

Relocation support should be architecturally possible even if automatic compaction is deferred.

---

30. V1 UPDATE PIPELINE

Connect:

dirty regions
      ↓
meshing requests
      ↓
worker threads
      ↓
completed meshes
      ↓
revision validation
      ↓
GPU uploads
      ↓
geometry allocation
      ↓
render metadata update

---

31. V1 STRESS TESTS

The implementation must test:

single edits
rapid random edits
boundary edits
continuous destruction
continuous construction
geometry growth
geometry shrinkage
allocation relocation
repeated free/reuse
stale mesh completion
large empty regions
large solid regions

Also create the earlier GPU baseline test of approximately 10,000 independently renderable simple regions/quads so CPU submission and visibility overhead can be measured consistently.

---

32. V1 DELIVERABLE

V1 is complete when this works reliably:

Lua or C++ modifies voxel
          ↓
WorldState revision changes
          ↓
VGP invalidates region
          ↓
meshing job executes
          ↓
revision is validated
          ↓
persistent geometry changes
          ↓
updated world becomes visible

Required output:

working executable
editable voxel world
naive mesher
greedy mesher
persistent geometry heap
LuaJIT mod capable of editing world
tests
profiling
stress benchmark

---

V2 — GPU-DRIVEN VISIBILITY

33. V2 OBJECTIVE

Stop requiring the CPU to submit every visible voxel region individually.

Target:

Persistent Geometry
        ↓
Render Metadata
        ↓
GPU Visibility
        ↓
Indirect Commands
        ↓
Renderer

---

34. V2 RENDER METADATA

Create compact GPU-visible metadata.

Conceptually:

bounds
geometry location
index count
material grouping
LOD identifier
flags

Keep it independent from WorldState storage.

---

35. V2 CPU VISIBILITY ORACLE

Implement/retain CPU frustum culling.

This remains the reference implementation against which GPU results are validated.

---

36. V2 GPU FRUSTUM CULLING

Implement flat GPU region culling first.

region metadata
      ↓
compute visibility
      ↓
visible list

Do not begin with a complicated hierarchical GPU implementation.

---

37. V2 INDIRECT RENDERING

Generate GPU draw commands from visible geometry.

GPU visibility
      ↓
command generation
      ↓
indirect draw buffer
      ↓
render

Measure CPU submission reduction.

---

38. V2 DISTANCE / SCREEN-SIZE REJECTION

Add inexpensive rejection criteria where useful.

Maintain the distinction:

Culling = should it render?

LOD = what representation should render?

---

39. V2 HIERARCHICAL GPU CULLING GATE

Do not automatically implement hierarchical GPU culling.

First profile flat GPU culling.

Implement hierarchical GPU traversal only if measurements demonstrate that testing all resident regions has become a significant cost.

The CPU VSH still exists regardless.

---

40. V2 OCCLUSION PROTOTYPE

Introduce conservative occlusion only after frustum/indirect rendering is stable.

False-positive visibility is acceptable.

False-negative visibility is not.

---

41. V2 ALLOCATOR HARDENING

Add:

page growth
fragmentation metrics
relocation
optional compaction
upload batching
retirement diagnostics

Compaction must not invalidate stable geometry identities.

---

42. V2 DELIVERABLE

V2 is complete when:

persistent geometry
      ↓
GPU visibility
      ↓
GPU command generation
      ↓
indirect rendering

operates without CPU per-region draw submission.

Required evidence:

CPU/GPU visibility comparison
CPU submission benchmark
GPU culling benchmark
allocator fragmentation benchmark
edit-to-visible latency benchmark
p50/p95/p99 frame timings

---

V3 — LOD + LARGE-WORLD STREAMING

43. V3 OBJECTIVE

Expand from an editable rendered world into a scalable hierarchical world.

Primary systems:

VSH
 +
LOD
 +
Streaming
 +
multiple geometry representations

---

44. V3 MULTI-RESOLUTION VSH

Use the existing hierarchy for meaningful coarse spatial decisions.

Expose query modes:

exact
sparse
coarse

Exact descends to the required finest representation.

Sparse intentionally stops earlier.

Coarse uses high hierarchy levels.

---

45. V3 SPATIAL QUERY API

Expose queries rather than raw VSH nodes.

Conceptually:

query_region()
query_occupancy()
trace_exact()
trace_sparse()
trace_coarse()

Additional volume queries may be added as concrete consumers require them.

This protects callers from future internal VSH representation changes.

---

46. V3 LOD POLICY

Create a dedicated LOD policy.

Inputs can include:

projected screen size
distance
current LOD
hysteresis
residency
visibility

Do not make meshing algorithms choose their own LOD.

---

47. V3 LOD HYSTERESIS

Avoid representation thrashing around thresholds.

Conceptually:

LOD 0 → LOD 1 threshold
        ≠
LOD 1 → LOD 0 threshold

This is especially important when geometry generation is asynchronous.

---

48. V3 LOD MESHER

Implement reduced-resolution geometry after the LOD policy itself is working.

Pipeline:

VGP
 ↓
desired LOD
 ↓
MeshingRequest
 ↓
WorldView
 ↓
LOD Mesher
 ↓
MeshData

---

49. V3 SURFACE EXTRACTION

If smooth voxel/signed-field terrain is part of the target representation, introduce the surface mesher after discrete LOD generation is stable.

Keep it behind the same meshing abstraction.

---

50. V3 TRANSITION GEOMETRY

Only after adjacent LOD levels render correctly independently should transition geometry be introduced.

Pipeline:

LOD A
  │
boundary
  │
LOD B
  ↓
transition request
  ↓
transition mesher

This isolates transition debugging from LOD-generation debugging.

---

51. V3 STREAMING MANAGER

Streaming owns residency.

spatial demand
      ↓
Streaming Manager
      ↓
IO
      ↓
decompression/deserialization
      ↓
WorldState residency

VGP may recommend demand.

It does not perform IO.

---

52. V3 STREAMING PRIORITY

Demand may eventually consider:

distance
visibility
screen importance
camera velocity
view direction
requested LOD
current residency

Use measured workloads to tune weighting.

---

53. V3 PREDICTIVE STREAMING

Use movement and view direction to request likely future regions before they are immediately needed.

Always preserve explicit memory/residency budgets.

---

54. V3 GEOMETRY RESIDENCY

A region may have:

no geometry
coarse geometry
fine geometry
transition geometry
multiple temporarily overlapping representations

VGP tracks the logical association.

Geometry Manager/Allocator owns physical storage.

---

55. V3 HIERARCHICAL GPU CULLING

If V2 profiling justified it, mirror the required portion of the hierarchy to the GPU and perform coarse-to-fine visibility.

root
 ↓
reject / descend
 ↓
reject / descend
 ↓
renderable leaves

If flat GPU culling remains inexpensive, this optimization may remain deferred.

---

56. V3 DELIVERABLE

V3 must demonstrate a world substantially larger than the immediately resident high-detail area.

Required behavior:

camera moves
     ↓
streaming predicts demand
     ↓
world data becomes resident
     ↓
VGP requests appropriate LOD
     ↓
geometry is generated
     ↓
allocator establishes residency
     ↓
GPU renders appropriate representation

Required deliverables:

multi-resolution VSH
LOD policy
LOD hysteresis
LOD meshing
surface-mesher architecture
transition geometry where required
streaming manager
predictive streaming
explicit residency budgets
exact/sparse/coarse queries
large-world benchmark

---

V4 — GPU SPATIAL PIPELINE + ADVANCED RENDERING

57. V4 OBJECTIVE

Use the mature spatial architecture for broader GPU-driven rendering and spatial queries.

VGP now becomes a reusable spatial/geometry service for multiple renderer features.

---

58. GPU VSH REPRESENTATION

Create a compact GPU representation containing only information needed for GPU traversal.

Do not blindly duplicate the complete CPU structure.

---

59. INCREMENTAL GPU VSH UPDATE

Use the existing dirty-region system.

WorldChange
    ↓
CPU VSH update
    ↓
changed hierarchy ranges
    ↓
GPU partial update

Local modifications should normally produce local transfers.

---

60. GPU SPATIAL TRAVERSAL

Implement coarse-to-fine traversal for workloads demonstrated to benefit from it.

Primary optimization:

EMPTY subtree
      ↓
skip entire spatial region

---

61. GPU LOD SELECTION

Move appropriate representation-selection work onto the GPU.

The CPU remains responsible for ensuring the required representations/resources exist.

GPU selection must not imply that the GPU performs meshing or storage IO.

---

62. GPU STREAMING FEEDBACK

The GPU may emit demand feedback:

GPU visibility
      ↓
missing representation
      ↓
request buffer
      ↓
CPU readback
      ↓
Streaming Manager

The CPU remains responsible for residency decisions and IO.

---

63. RENDERER SPATIAL QUERIES

Evaluate VGP/VSH traversal for:

shadow visibility
ambient occlusion
reflection candidates
coarse environmental visibility

Choose exact, sparse or coarse traversal according to quality requirements.

---

64. STOCHASTIC QUERYING

Where appropriate:

surface
 ↓
stochastic sample
 ↓
VSH traversal
 ↓
approximate spatial result

The spatial system supplies the query.

The renderer owns sampling strategy.

---

65. TEMPORAL RECONSTRUCTION

Renderer responsibility:

current sample
      +
previous history
      +
motion/reprojection
      ↓
temporal accumulation
      ↓
filter/reconstruction

Do not place temporal state inside VGP.

---

66. MATERIAL SUMMARIES

Only add hierarchical material classification if a concrete query benefits.

Potential conservative summaries:

EMPTY
OPAQUE
TRANSPARENT
LIQUID
EMISSIVE
MIXED

Avoid turning VSH into another material database.

---

67. LUAJIT MOD API MATURATION

By V4 the public mod API should be explicitly versioned.

Mods interact with stable services:

World
Entities
Events
Assets
Content
Configuration
Procedural generation
Tools

Internal C++ architecture may evolve independently of the public mod API.

---

68. V4 DELIVERABLE

V4 should demonstrate:

editable large world
       ↓
incremental CPU spatial updates
       ↓
incremental GPU spatial updates
       ↓
GPU visibility + LOD
       ↓
indirect rendering
       +
advanced spatial queries
       ↓
advanced renderer

Required deliverables:

GPU VSH
incremental synchronization
GPU traversal
GPU-assisted LOD
GPU streaming feedback
exact/sparse/coarse renderer queries
spatially assisted shadows/AO/reflections where validated
stochastic rendering where useful
temporal reconstruction
mature versioned mod API
production diagnostics

---

69. PERMANENT CORRECTNESS ORACLES

Do not delete simple implementations merely because optimized implementations exist.

Retain in development/test configurations:

Naive Mesher
CPU exact VSH traversal
CPU frustum culling
simple direct draw path
allocator validation

These provide independent references for optimized systems.

---

70. PROFILING REQUIREMENTS

Every performance test should capture at least:

CPU frame time
GPU frame time
p50 frame time
p95 frame time
p99 frame time
worst observed frame
WorldState update time
VSH update time
meshing latency
meshing throughput
GPU upload volume
edit-to-visible latency
resident geometry memory
allocator fragmentation
visible regions
culled regions
indirect command count
streaming bandwidth
streaming latency

Do not optimize from averages alone.

Voxel editing and streaming systems are particularly sensitive to latency spikes.

---

71. CRITICAL BENCHMARK SUITE

Maintain reproducible scenes for:

empty world
solid world
checkerboard/high-surface-complexity world
10,000 independent renderables
large static terrain
rapid random voxel edits
localized destruction
large explosion/destruction
continuous construction
camera high-speed traversal
LOD threshold movement
streaming memory pressure
allocator fragmentation

Benchmark results should be comparable between commits.

---

72. AGENT PHASE REPORT

At every implementation milestone, the Agent must record:

IMPLEMENTED
what changed

OWNERSHIP
which subsystem owns it

DEPENDENCIES
new dependencies introduced

CORRECTNESS
tests performed

PERFORMANCE
measurements collected

LIMITATIONS
known incomplete behavior

ARCHITECTURE
whether any established boundary changed

NEXT STEP
next dependency in the roadmap

Architecture changes require explicit justification before implementation.

---

73. AGENT EXECUTION ORDER

The implementation order is:

01  Repository/build/toolchain
02  Core types/logging/assertions/errors
03  Math
04  Memory foundations
05  Platform/window/input/time
06  Test framework
07  Profiling framework
08  GPU abstraction
09  Primitive renderer
10  First executable vertical slice
11  Basic job system
12  LuaJIT embedding
13  Mod loader/public API foundation
14  WorldState
15  Region/revision model
16  WorldView
17  WorldChange notifications
18  VGP coordinator foundation
19  VSH L0 occupancy
20  VSH hierarchy
21  Incremental dirty propagation
22  Boundary invalidation
23  CPU exact traversal
24  Naive mesher
25  MeshData contract
26  Persistent paged GPU geometry allocator
27  World → VGP → meshing → allocator → renderer slice
28  Greedy mesher
29  Asynchronous meshing queues
30  Revision/stale-result rejection
31  Backpressure/coalescing
32  Mutation stress testing

--- V1 GATE ---

33  GPU render metadata
34  CPU visibility oracle
35  GPU flat frustum culling
36  GPU indirect command generation
37  Distance/screen-size rejection
38  Conservative occlusion prototype
39  Allocator relocation/fragmentation hardening
40  Profile hierarchical GPU culling requirement

--- V2 GATE ---

41  Multi-resolution spatial query API
42  Exact/sparse/coarse traversal
43  LOD policy
44  LOD hysteresis
45  LOD mesher
46  Surface mesher where required
47  LOD transition detection
48  Transition/Transvoxel implementation
49  Streaming manager
50  Streaming budgets
51  Streaming priority
52  Predictive streaming
53  Geometry residency management
54  Hierarchical GPU culling if profiling justified it

--- V3 GATE ---

55  Compact GPU VSH representation
56  Incremental CPU→GPU VSH synchronization
57  GPU hierarchy traversal
58  GPU-assisted LOD
59  GPU streaming feedback
60  Renderer spatial-query integration
61  Shadow-query prototype
62  AO-query prototype
63  Reflection-query prototype
64  Stochastic query/render experiments
65  Temporal reconstruction
66  Material summaries only where justified
67  Mature/versioned LuaJIT Mod API
68  Production profiling/debugging tools

--- V4 GATE ---

69  Full-system stress testing
70  Cross-system profiling
71  Optimize demonstrated bottlenecks
72  Production hardening

---

74. FINAL ARCHITECTURE

                           LuaJIT Mods
                               │
                        Stable Public API
                               │
                               ▼
                         ┌───────────┐
                         │WorldState │
                         └─────┬─────┘
                               │
                      WorldChange + revision
                               │
                               ▼
                    ┌────────────────────┐
                    │        VGP         │
                    │                    │
                    │  ┌──────────────┐  │
                    │  │     VSH      │  │
                    │  │ occupancy    │  │
                    │  │ hierarchy    │  │
                    │  │ traversal    │  │
                    │  └──────────────┘  │
                    │                    │
                    │ dirty tracking     │
                    │ work scheduling    │
                    │ LOD coordination   │
                    │ geometry mapping   │
                    └─────────┬──────────┘
                              │
                       MeshingRequest
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             ▼                ▼                ▼
           Naive            Greedy        LOD/Surface
             │                │                │
             └────────────────┼────────────────┘
                              │
                           MeshData
                              │
                              ▼
                    Geometry Manager
                              │
                              ▼
                  Persistent GPU Allocator
                              │
                              ▼
                       GPU Geometry
                              │
                  ┌───────────┴────────────┐
                  ▼                        ▼
             Render Metadata          GPU VSH
                  │                        │
                  └──────────┬─────────────┘
                             ▼
                    GPU Visibility / LOD
                             │
                             ▼
                     Indirect Commands
                             │
                             ▼
                         Renderer
                             │
                  ┌──────────┼───────────┐
                  ▼          ▼           ▼
               Lighting    Shadows    Reflections
                  │          │           │
                  └──────────┼───────────┘
                             ▼
                    Temporal / Stochastic
                       Reconstruction
                             │
                             ▼
                            Frame

---

75. CORE ARCHITECTURAL RULE

The entire engine can be summarized as:

WorldState owns truth.

VGP owns voxel-geometry coordination.

VSH owns derived hierarchical spatial knowledge.

Meshers convert authoritative voxel samples into geometry.

MeshData transports generated geometry.

Geometry Manager owns logical geometry resources.

Persistent GPU Allocator owns physical GPU geometry storage.

GPU visibility determines what should be drawn.

LOD determines which representation should be drawn.

Streaming determines what world/resources should be resident.

Renderer creates the frame.

LuaJIT extends the engine through stable public APIs.

No optimization should violate these ownership boundaries without a demonstrated architectural reason.This version also resolves what I think was the most important latent issue in the previous roadmap: the mesher no longer logically consumes VGP as its voxel-data source. VGP schedules and coordinates meshing; a read-only view of WorldState supplies the actual voxel samples. That distinction will matter enormously once meshing, edits, streaming and LOD are all asynchronous.

I also restored the stronger parts of our original rendering plan—particularly the persistent paged geometry heap, explicit dirty→mesh→upload queues, mutation stress tests, allocator relocation, GPU-safe retirement, and the requirement to profile flat GPU culling before adding hierarchical GPU culling. Those were worth preserving rather than allowing the later, broader engine roadmap to dilute them.