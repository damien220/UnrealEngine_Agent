# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A collection of **Claude Code agent definition files** — specialized sub-agent personalities for Unreal Engine 5 development. These markdown files are loaded by the Claude Code FleetView harness and define the identity, rules, deliverables, and communication style of each specialist agent.

## Directory Structure

```
UnrealEngine_Agent/
├── CLAUDE.md
├── README.md
├── plan.md                             ← enhancement roadmap and new agent proposals
├── unreal-systems-engineer.md          ← agent definitions at root (FleetView convention)
├── unreal-multiplayer-architect.md
├── unreal-world-builder.md
├── unreal-blueprint-specialist.md
└── refs/                               ← on-demand knowledge files, not pre-loaded
    ├── systems-engineer/               ← owned by UnrealSystemsEngineer (complete, 12 files)
    │   ├── blueprint-cpp-boundary.md   ← Tick rule, UFUNCTION macros, Blueprint scope
    │   ├── nanite-lumen.md             ← 16M limit, incompatibilities, Lumen setup, profiling
    │   ├── memory-gc.md                ← UPROPERTY rules, TWeakObjectPtr, IsValid()
    │   ├── gas-setup.md                ← .Build.cs deps, UAttributeSet, UGameplayAbility, tags
    │   ├── build-system.md             ← GenerateProjectFiles, circular deps, reflection macros
    │   ├── crash-prevention.md         ← 25-pattern crash audit with applied fixes
    │   ├── object-lifecycle.md         ← init phases, CreateDefaultSubobject vs NewObject
    │   ├── cpp-casting.md              ← Cast vs CastChecked vs static_cast, UInterface calls
    │   ├── architecture-patterns.md    ← composition, delegates, subsystems, object pooling
    │   ├── threading-async.md          ← game thread rule, AsyncTask, FCriticalSection
    │   ├── asset-management.md         ← hard vs soft refs, TSoftObjectPtr, FStreamableManager
    │   └── logging-assertions.md       ← log categories, check/ensure semantics, CVars
    ├── multiplayer-architect/          ← owned by UnrealMultiplayerArchitect (complete, 7 files)
    │   ├── authority-rpcs.md           ← server authority, RPC taxonomy, Reliable vs Unreliable
    │   ├── replication-efficiency.md   ← DOREPLIFETIME conditions, NetUpdateFrequency, dormancy
    │   ├── network-hierarchy.md        ← GameMode/GameState/PlayerState/PlayerController layout
    │   ├── gas-replication.md          ← dual init path, ASC replication modes, prediction
    │   ├── dedicated-server.md         ← server target, RunUAT, DefaultGame.ini, profiling
    │   ├── anti-cheat.md               ← _Validate pattern, exploit categories, audit logging
    │   └── advanced-replication.md     ← Replication Graph, Network Prediction Plugin
    ├── world-builder/                  ← planned for Phase 3
    └── blueprint-specialist/           ← owned by UnrealBlueprintSpecialist (complete, 5 files)
        ├── graph-organization.md       ← container types, 15-node threshold, naming conventions
        ├── communication-patterns.md   ← Interface vs Dispatcher vs Cast decision tree
        ├── performance-patterns.md     ← VM cost model, Tick discipline, GetAllActors caching
        ├── cpp-handoff.md              ← UFUNCTION/UPROPERTY macros, metadata, UINTERFACE
        └── data-assets.md              ← PrimaryDataAsset, DataTable, Asset Manager, soft refs
```

Agent `.md` files stay at root — FleetView scans the root directory for agent definitions. The `refs/` subtree is knowledge-on-demand: agents read only the file relevant to the current task, keeping per-invocation context minimal.

## Agent Files

Each root `.md` file defines one agent:

| File | Agent Name | Specialty |
|------|-----------|-----------|
| `unreal-systems-engineer.md` | UnrealSystemsEngineer | C++/Blueprint architecture, GAS setup, Nanite/Lumen, memory model, tick optimization |
| `unreal-multiplayer-architect.md` | UnrealMultiplayerArchitect | Actor replication, GameMode/GameState hierarchy, GAS networking, dedicated server |
| `unreal-world-builder.md` | UnrealWorldBuilder | World Partition, Landscape, PCG foliage, HLOD, large-scale streaming |
| `unreal-blueprint-specialist.md` | UnrealBlueprintSpecialist | Blueprint graph organization, communication patterns, performance discipline, C++ handoff |

## Reference Files (`refs/`)

Ref files are topic-scoped companion documents shipped alongside their agent. They are registered in the agent's `## 🚨 Critical Rules` section with a 2–3 sentence summary and a `Read refs/... when:` trigger. The agent reads the file only when the task falls in that area — never pre-emptively. This keeps the main agent context small while providing full depth when needed.

**Design principle**: agent files define identity and workflow; ref files define knowledge. Updating a ref file — to fix a rule, add examples, or reflect a new UE5 version — never requires touching the agent.

## Agent File Format

Each agent file uses YAML frontmatter followed by a structured markdown body:

```yaml
---
name: Agent Display Name
description: One-line description shown in FleetView
color: red | orange | green | blue | ...
emoji: 🌐
vibe: Short tagline shown in UI
---
```

The markdown body follows this section structure (in order):

1. `# <AgentName> Agent Personality` — H1 title
2. `## 🧠 Your Identity & Memory` — role, personality, experience framing
3. `## 🎯 Your Core Mission` — primary goals with bullet list
4. `## 🚨 Critical Rules You Must Follow` — hard constraints grouped by topic (most important section — each topic is a 2–3 sentence summary + `Read refs/... when:` trigger)
5. `## 📋 Your Technical Deliverables` — canonical code examples with comments
6. `## 🔄 Your Workflow Process` — numbered step-by-step process
7. `## 💭 Your Communication Style` — tone and framing examples
8. `## 🎯 Your Success Metrics` — measurable pass/fail criteria
9. `## 🚀 Advanced Capabilities` — expert-level extensions (optional)

## Authoring Guidelines

### Critical Rules section
- This section determines correctness — spend the most effort here
- Each topic: one `### Heading`, 2–3 sentence summary, then a `Read refs/... when:` trigger line
- The summary must be specific enough to act as a standalone reminder — not just "see the ref file"
- The trigger condition must be narrow: one sentence listing the concrete task signals that warrant loading the file

### Adding a New Rule
- If the rule is narrow and needs no code example: add a bullet to the relevant ref file
- If it requires code, tables, or more than 5 lines: create a new ref file and register it in the agent
- If it applies to multiple agents (e.g. `IsValid()` over `!= nullptr`): update all affected agent files

### Technical Deliverables section
- Show complete, copy-paste-ready C++ code blocks, not pseudocode
- Comments explain the UE5 *why*, not just the what
- Each block has a clear heading explaining what it demonstrates

### Success Metrics section
- Make metrics quantitative where possible: "< 15KB/s per player", "16M instance limit", "< 1 per 30 seconds"
- Frame as pass/fail criteria the agent itself can evaluate

### Cross-agent consistency
The three agents share overlapping UE5 concepts. Keep these consistent across files:
- **Nanite instance limit**: 16 million — cited the same way in all agents
- **GAS module dependencies**: `GameplayAbilities`, `GameplayTags`, `GameplayTasks` — same in all agents
- **`IsValid()` over `!= nullptr`** — enforced in SystemsEngineer, respected in the others
- **Network hierarchy** (GameMode/GameState/PlayerState/PlayerController) — defined in MultiplayerArchitect, referenced consistently elsewhere

## Development Status

### Phase 1 — UnrealSystemsEngineer (complete)

All 12 ref files created. The agent's `Critical Rules` section uses the full summary+trigger pattern throughout — no inline rule content remains in the main file.

| Ref file | Coverage |
|---|---|
| `blueprint-cpp-boundary.md` | Tick rule, data types, engine extensions, UFUNCTION macros, Blueprint scope |
| `nanite-lumen.md` | 16M instance limit, incompatibility table, Lumen setup, profiling commands |
| `memory-gc.md` | UPROPERTY rules, TWeakObjectPtr, TSharedPtr, IsValid(), TObjectPtr, GC timing |
| `gas-setup.md` | .Build.cs deps, ASC replication modes, UAttributeSet, UGameplayAbility, tag system |
| `build-system.md` | GenerateProjectFiles triggers, module structure, circular deps, reflection macros |
| `crash-prevention.md` | 25-pattern crash audit with code-level fixes |
| `object-lifecycle.md` | Init phases, CDO trap, CreateDefaultSubobject vs NewObject vs SpawnActorDeferred |
| `cpp-casting.md` | Cast vs CastChecked vs static_cast, UInterface calling conventions |
| `architecture-patterns.md` | Composition, delegates, subsystems, object pooling |
| `threading-async.md` | Game thread rule, AsyncTask, FCriticalSection, ParallelFor |
| `asset-management.md` | Hard vs soft refs, TSoftObjectPtr, FStreamableManager |
| `logging-assertions.md` | Log categories, check/ensure semantics, CVars, DrawDebug stripping |

---

### Phase 2 — UnrealMultiplayerArchitect (complete)

All 7 ref files created. The agent's `Critical Rules` section uses the full summary+trigger pattern throughout — no inline rule content remains in the main file. Step 0 (code-mapper project structure) added to Workflow.

| Ref file | Coverage |
|---|---|
| `authority-rpcs.md` | Server authority model, UFUNCTION taxonomy (Server/Client/NetMulticast), Reliable vs Unreliable, RPC ordering, ownership rules |
| `replication-efficiency.md` | DOREPLIFETIME conditions, NetUpdateFrequency per actor class, GetNetPriority, actor dormancy, RepNotify patterns |
| `network-hierarchy.md` | GameMode/GameState/PlayerState/PlayerController — what each owns, replication scope, access patterns, violation table |
| `gas-replication.md` | Dual init path (PossessedBy + OnRep_PlayerState), ASC replication modes, prediction (LocalPredicted/ServerInitiated), FPredictionKey, FGameplayEffectContext |
| `dedicated-server.md` | Server target file, RunUAT build command, DefaultGame.ini config, IsRunningDedicatedServer skips, bandwidth config, profiling, graceful shutdown, Online Beacons |
| `anti-cheat.md` | _Validate pattern, distance/rate/range exploits, re-check in _Implementation, audit logging, security checklist |
| `advanced-replication.md` | Replication Graph plugin, GridSpatialization2D, frequency bucket nodes, Network Prediction Plugin state types and simulation callback, NPP vs CMC comparison |

---

### Phase 3 — UnrealWorldBuilder

Apply the same ref-file structure to `unreal-world-builder.md`.

| Topic | Planned ref file |
|---|---|
| World Partition grid & data layers | `refs/world-builder/world-partition.md` |
| Landscape resolution, layers & RVT | `refs/world-builder/landscape.md` |
| HLOD configuration & rebuild workflow | `refs/world-builder/hlod.md` |
| PCG graph design & exclusion zones | `refs/world-builder/pcg-foliage.md` |
| Streaming performance & profiling checklist | `refs/world-builder/streaming-performance.md` |
| Large World Coordinates & OFPA | `refs/world-builder/advanced-world.md` |

---

### Phase 4 — UnrealBlueprintSpecialist (complete)

Designer-facing C++/Blueprint continuum: graph organization, communication patterns, performance discipline, C++ handoff macros, and data assets.

| Ref file | Coverage |
|---|---|
| `refs/blueprint-specialist/graph-organization.md` | Container types, 15-node extraction threshold, Pure vs Non-Pure evaluation, naming conventions, compilation size thresholds |
| `refs/blueprint-specialist/communication-patterns.md` | Interface vs Dispatcher vs Direct cast decision tree, unbind discipline, Level Blueprint restrictions |
| `refs/blueprint-specialist/performance-patterns.md` | Blueprint VM cost model, Tick discipline, GetAllActorsOfClass caching, ForEach C++ migration, Blueprint Profiler usage |
| `refs/blueprint-specialist/cpp-handoff.md` | UFUNCTION macro table, UPROPERTY specifiers, metadata reference table, Blueprint Function Library, UINTERFACE |
| `refs/blueprint-specialist/data-assets.md` | PrimaryDataAsset/DataTable/CurveTable choice table, Asset Manager registration, soft vs hard reference rules |

See `plan.md` for the full enhancement roadmap and new agent proposals.

---

## UE5 Quick-Reference Constants

- Nanite max instances per scene: **16 million**
- LWC required for worlds: **> 2km** in any axis (floating point errors visible at ~20km without it)
- Landscape max visible layers per region: **4**
- Default reasonable enemy tick rate: **20Hz** (not the default 60+)
- GAS build dependencies: `"GameplayAbilities"`, `"GameplayTags"`, `"GameplayTasks"` in `PublicDependencyModuleNames`
- World Partition cell sizes: 32–64m urban, 128m open terrain, 256m+ sparse areas
- HLOD required for geometry visible at: **> 500m**
- PCG pre-bake required for zones: **> 1km²**
