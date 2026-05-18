# Blueprint Communication Patterns — UE5 Reference

## The Three Patterns and When to Use Each

| Pattern | Coupling | Directionality | Best for |
|---|---|---|---|
| **Blueprint Interface** | Decoupled — caller does not know receiver's type | One sender → one receiver (or broadcast via `GetAllActors`) | Cross-type messaging where sender should not know receiver's class |
| **Event Dispatcher** | Decoupled — broadcaster does not know subscribers | One broadcaster → many subscribers | Notification systems where 0–N objects react to an event |
| **Direct Reference / Cast** | Tightly coupled — caller knows receiver's concrete type | One caller → one specific receiver | Owner–child relationships where coupling is intentional and correct |

**The decision question:** Does the caller need to know the concrete type of the receiver?
- No → Interface or Dispatcher
- Yes, one-to-many → Dispatcher
- Yes, specific known type → Direct reference

---

## Blueprint Interfaces

Use when the caller must send a message to an object whose concrete type it should not know:

```
Scenario: A Trigger Volume wants to tell the Actor that entered it to "interact".
The Actor might be a door, a chest, a character, an elevator.
The Trigger does not know — and should not care — which one.

Solution: IMyInteractable interface.
The Trigger calls Execute_Interact on the overlapping actor.
Only Actors that implement the interface respond.
```

### Calling an Interface in Blueprint

1. Get a reference to the target Actor
2. Use the `Does Implement Interface` node to check (optional — call is silently dropped if not implemented)
3. Call the interface message node (appears in the palette as `[Interface Name] Message: [FunctionName]`)

### Interface vs Cast Decision Tree

```
Do I own this reference (parent → child, owner → component)?
  Yes → Direct reference. Cast is fine.
  No → Does the sender need a return value from the receiver?
    Yes → Interface with return value.
    No → Does 1 event go to N unknown receivers?
      Yes → Event Dispatcher.
      No → Interface.
```

**Cast chains are always wrong:**
```
BAD:
[Cast to ADoor] → on fail → [Cast to AChest] → on fail → [Cast to AElevator]

GOOD:
[Does Implement Interface: IMyInteractable?] → [Execute Interact]
```

---

## Event Dispatchers

Use when one object needs to announce an event and zero or more other objects should react — without the broadcaster knowing who they are:

```
Scenario: A Health Component loses all HP.
The Character, the HUD, the Achievement System, and the Round Manager all need to react.
The Health Component should not hold references to all four.

Solution: OnDied dispatcher on the Health Component.
Each interested object binds to OnDied in their BeginPlay.
```

### Dispatcher Binding Best Practices

**Always unbind in EndPlay or on component destruction:**
```
[Event BeginPlay]
  → [Get Target Actor's Health Component]
  → [Bind Event to OnDied]
    → [Custom Event: Handle_OnDied]

[Event EndPlay]
  → [Get Target Actor's Health Component]
  → [Unbind Event from OnDied]  ← Never skip this — stale bindings crash
```

Failing to unbind causes the subscriber to fire after its owning actor is destroyed — a common crash source.

### Dispatcher vs Interface for Broadcast

| | Event Dispatcher | Interface + GetAllActors |
|---|---|---|
| Performance | O(subscriber count) — only bound objects notified | O(world actors) — iterates entire world |
| Registration | Explicit bind required | Implicit — all implementing actors respond |
| Use when | Subscribers are known at BeginPlay | Truly dynamic — unknown count, can't bind at spawn |

`GetAllActorsWithInterface` is O(world) — use it only for one-time setup calls, never in Tick or per-event logic.

---

## Direct References and Cast Discipline

Direct casts are correct in exactly two situations:

1. **Owner→child relationship**: An Actor casts its own component (`Cast To UMyHealthComponent` to get its own component)
2. **Known-type callback**: A delegate callback that you know will only ever receive one concrete type (e.g., `OnPossess` in your own Character subclass)

```
CORRECT — Actor accesses its own component
[Get Component by Class: UMyHealthComponent] → [Cast To UMyHealthComponent]

WRONG — Trigger Volume casts overlapping actor to a specific type
[On Component Begin Overlap] → [Cast To AMyDoor]
  → (brittle: breaks if the actor type changes)
  → (fix: use IMyInteractable interface)
```

---

## Level Blueprint Communication

Level Blueprint should only communicate with persistent level actors for map-specific scripting (cutscene triggers, tutorial flows). It must never hold persistent references to spawned actors — spawned actors are destroyed on travel or respawn, leaving the Level Blueprint with dangling references.

```
CORRECT use:
Level Blueprint binds to a static PlacedTrigger actor's dispatcher
→ starts a cinematic when triggered

WRONG use:
Level Blueprint holds a reference to a spawned Character
→ Character dies and respawns → Level Blueprint reference is stale → crash
```

For persistent game communication, use `GameInstance`, `GameState`, or a `UGameInstanceSubsystem` — not Level Blueprint.

---

## Object Communication Summary Table

| From | To | Pattern |
|---|---|---|
| Actor → its own component | Direct reference | `GetComponentByClass` + Cast |
| Actor → unknown Actor type | Interface | `Execute_[FunctionName]` |
| Component → owner's response | `BlueprintImplementableEvent` on owner | Define in C++, author in BP |
| System → all interested parties | Event Dispatcher | Bind in BeginPlay, Unbind in EndPlay |
| UI Widget → Game logic | Interface on HUD or GameMode | Avoids direct game reference from UI |
| Game logic → UI Widget | Direct reference (HUD owns widget) | Cast to specific widget class |
