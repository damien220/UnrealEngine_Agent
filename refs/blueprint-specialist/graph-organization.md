# Blueprint Graph Organization — UE5 Reference

## The Fundamental Rule

The Event Graph is a router, not an implementation. It receives events and delegates to named Functions. Any chain exceeding 15 nodes or branching more than twice belongs in a Function with a descriptive name, not inline in the Event Graph.

---

## Container Types and When to Use Each

| Container | Compilation unit | Debuggable | Reusable | Side effects | When to use |
|---|---|---|---|---|---|
| **Function** | Separate call site | Yes — breakpoint support | Yes — call from multiple graphs | Allowed | Any logic > 3 nodes; all named behavior |
| **Macro** | Inlined at call site | No — executes as part of caller | Across graphs in same Blueprint | Allowed | Repeated node patterns within one Blueprint only |
| **Collapsed Graph** | Inlined at call site | No | No — local to that graph | Allowed | Temporary tidying only; extract to Function before shipping |
| **Event Graph** | Root entry points | Yes | No | Required | Event routing only — receive event, call Functions |

**Collapsed Graphs do not reduce compilation size** — they are visual compression only. Extract to Functions if the logic is non-trivial.

---

## Function Extraction Threshold

Extract to a named Function when any of these are true:
- Node chain exceeds **15 nodes**
- Logic branches more than **twice** (three separate paths)
- The same subgraph appears in **more than one place**
- The logic has a name you can state in a sentence (it has semantic identity)

```
BAD — Event Graph:
[Event BeginPlay] → [Get Health Component] → [Is Valid] → [Branch] → [Get Max Health]
→ [Set Health Bar Max] → [Get Current Health] → [Divide] → [Set Health Bar Percent]
→ [Get Shield Component] → [Is Valid] → [Branch] → [Get Shield Value] → [Set Shield Bar]

GOOD — Event Graph delegates to named Functions:
[Event BeginPlay] → [Initialize Health Bar] → [Initialize Shield Bar]

Each Function contains its own 6-8 node chain with a clear name.
```

---

## Pure vs Non-Pure Functions

| Type | Execution pin | Evaluated when | Use for |
|---|---|---|---|
| Non-Pure (default) | Yes | Explicit execution flow | Functions with side effects; anything that modifies state |
| Pure (`BlueprintPure`) | No | Every time output is read | Stateless getters, math helpers, value conversions |

**Pure functions execute every time their output wire is read.** If a Pure function is called by 3 nodes, it runs 3 times per frame. Cache results in a local variable if the Pure function is expensive.

```
BAD — Pure function called 4 times:
[Get Player Health] (Pure) → [> 0] branch condition
[Get Player Health] (Pure) → [/ Max Health] for health bar
[Get Player Health] (Pure) → [<= 20] for danger sound
[Get Player Health] (Pure) → [Format Text] for HUD display

GOOD — Call once, cache:
[Get Player Health] (Non-Pure) → [Set Local Var: CurrentHealth]
[CurrentHealth] → [> 0] branch
[CurrentHealth] → [/ Max Health] for health bar
...
```

---

## Naming Conventions

All functions, variables, and events follow `[Verb][Noun]` or `[Adjective][Noun]` convention:

| Type | Convention | Example |
|---|---|---|
| Functions (action) | `[Verb][Noun]` | `UpdateHealthBar`, `InitializeInventory`, `PlayDeathMontage` |
| Functions (query) | `Get[Noun]`, `Is[State]`, `Has[Noun]` | `GetCurrentHealth`, `IsAlive`, `HasActiveShield` |
| Events | `On[Noun][Verb Past]` | `OnActorDied`, `OnItemPickedUp`, `OnRoundEnded` |
| Variables (state) | `[Noun]` or `b[State]` for bools | `CurrentHealth`, `bIsSprintActive`, `EquippedWeapon` |
| Local variables | `[noun]` lowercase | `healthPercent`, `targetActor` |
| Dispatchers | `On[Noun][Verb Past]` | `OnHealthChanged`, `OnInventoryFull` |

---

## Blueprint Compilation Size

Each Function in a Blueprint adds to compiled bytecode. A Blueprint with hundreds of Functions becomes slow to compile and large in memory. Signs that a Blueprint has grown too large:

- Editor compile time > 5 seconds
- Blueprint file size > 2MB on disk
- > 50 Functions in a single Blueprint class

When this occurs, split the Blueprint: extract subsystems into Actor Components (`UActorComponent`) with their own Blueprints, then compose them on the parent Actor.

---

## Comments and Documentation Nodes

Comment boxes in Blueprint are zero-cost at runtime (stripped on compile). Use them for every logical section:

- Color-code by concern: input handling (blue), damage logic (red), animation (green), UI (yellow)
- Every Function with non-obvious inputs gets a tooltip comment on the entry node
- Every Event Dispatcher binding site gets a comment explaining what it listens for and why

---

## Graph Layout Discipline

- Left-to-right execution flow — entry nodes on the left, terminal nodes on the right
- No crossed wires between unrelated execution chains
- Separate chains by at least 3 grid units of vertical space
- Reroute nodes (`R` key) instead of long diagonal wires across the graph
