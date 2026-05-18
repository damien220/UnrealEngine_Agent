# Unreal Engine Agent Collection

A collection of specialized [Claude Code](https://claude.ai/code) sub-agent definitions for Unreal Engine 5 development. Each agent is a markdown file loaded by the Claude Code FleetView harness — giving it a distinct identity, rule set, and deep knowledge for a specific UE5 domain.

---

## Why This Exists

Working on UE5 with a general-purpose AI assistant means re-explaining the same context in every session: that `UPROPERTY()` is required for garbage collection, that Blueprint Tick is a per-frame VM cost, that GAS needs three specific module dependencies, that Nanite has a 16-million instance ceiling per scene. These agents encode that context permanently.

Each agent is a specialist that already assumes AAA-grade UE5 conventions. Pick the one that matches your task and get answers without re-establishing ground rules every time.

---

## Agents

| Agent                             | Specialty                                                                                        | Status                                 |
| --------------------------------- | ------------------------------------------------------------------------------------------------ | -------------------------------------- |
| `unreal-systems-engineer.md`      | C++/Blueprint architecture, GAS setup, Nanite/Lumen, memory model, tick optimization             | Complete — 12 ref files                |
| `unreal-multiplayer-architect.md` | Actor replication, GameMode/GameState hierarchy, server-authoritative gameplay, dedicated server | Complete — 7 ref files                 |
| `unreal-world-builder.md`         | World Partition, Landscape, PCG foliage, HLOD, large-scale streaming                             | Agent complete — ref files in progress |
| `unreal-blueprint-specialist.md`  | Blueprint graph organization, communication patterns, performance discipline, C++ handoff        | Complete — 5 ref files                 |

---

## How It Works

### Agent Files

Each root `.md` file defines one specialist agent using YAML frontmatter followed by a structured markdown body:

```yaml
---
name: Unreal Systems Engineer
description: Performance and hybrid architecture specialist
color: orange
emoji: ⚙️
vibe: Masters the C++/Blueprint continuum for AAA-grade Unreal Engine projects.
---
```

The body is organized into fixed sections:

| Section                     | Purpose                                                        |
| --------------------------- | -------------------------------------------------------------- |
| `## Identity & Memory`      | Role, personality, experience framing                          |
| `## Core Mission`           | Primary goals — what the agent optimizes for                   |
| `## Critical Rules`         | Hard constraints grouped by topic — the most important section |
| `## Technical Deliverables` | Copy-paste-ready C++ and config code blocks                    |
| `## Workflow Process`       | Step-by-step process the agent follows                         |
| `## Communication Style`    | Tone and how the agent explains tradeoffs                      |
| `## Success Metrics`        | Quantitative pass/fail criteria                                |
| `## Advanced Capabilities`  | Expert-level extensions (optional)                             |

### Reference Files

The `refs/` directory holds topic-scoped knowledge files that agents load **on demand** — never pre-loaded into every session. Each topic in an agent's `Critical Rules` section is a 2–3 sentence summary plus a `Read refs/... when:` trigger line. The agent reads the full ref file only when the task falls into that area.

This keeps per-session context small while providing full technical depth when needed. Updating a ref file (to fix a rule, add examples, or reflect a new UE5 release) never requires touching the agent's personality or workflow.

```
refs/
├── systems-engineer/       (12 files — complete)
├── multiplayer-architect/  (7 files — complete)
├── world-builder/          (planned)
└── blueprint-specialist/   (5 files — complete)
```

---

## Usage

These files are used with the [Claude Code](https://claude.ai/code) FleetView harness. In FleetView, each agent appears as a selectable specialist. Pick the one that matches your current task:

- Building a gameplay system, GAS ability, or C++ architecture → **Unreal Systems Engineer**
- Setting up multiplayer, replication, or a dedicated server → **Unreal Multiplayer Architect**
- Working on open-world levels, landscape, or streaming → **Unreal World Builder**
- Authoring Blueprint graphs, communication patterns, or data assets → **Unreal Blueprint Specialist**

The agent is aware of its own ref files. When you ask a question in a ref file's scope, it reads that file before answering — you do not need to reference it yourself.

---

## Project Structure

```
UnrealEngine_Agent/
├── README.md
├── CLAUDE.md                               ← authoring guidelines for contributors
├── unreal-systems-engineer.md
├── unreal-multiplayer-architect.md
├── unreal-world-builder.md
├── unreal-blueprint-specialist.md
└── refs/
    ├── systems-engineer/
    │   ├── blueprint-cpp-boundary.md
    │   ├── nanite-lumen.md
    │   ├── memory-gc.md
    │   ├── gas-setup.md
    │   ├── build-system.md
    │   ├── crash-prevention.md
    │   ├── object-lifecycle.md
    │   ├── cpp-casting.md
    │   ├── architecture-patterns.md
    │   ├── threading-async.md
    │   ├── asset-management.md
    │   └── logging-assertions.md
    ├── multiplayer-architect/
    │   ├── authority-rpcs.md
    │   ├── replication-efficiency.md
    │   ├── network-hierarchy.md
    │   ├── gas-replication.md
    │   ├── dedicated-server.md
    │   ├── anti-cheat.md
    │   └── advanced-replication.md
    └── blueprint-specialist/
        ├── graph-organization.md
        ├── communication-patterns.md
        ├── performance-patterns.md
        ├── cpp-handoff.md
        └── data-assets.md
```

---

## Roadmap

This project is under active, continuous development. The architecture is designed to grow incrementally — new ref files can be added to any agent without touching its core definition, and new agents can be added alongside existing ones.

**In progress:**

- Ref file coverage for `unreal-world-builder.md` — World Partition grid, Landscape layers, HLOD workflow, PCG foliage, streaming performance, Large World Coordinates

**Planned — agent enhancements:**

- Unreal Systems Engineer: shader/material system, Enhanced Input migration, save system patterns
- Unreal Multiplayer Architect: session and matchmaking, voice chat, lag compensation patterns
- Unreal Blueprint Specialist: UMG/Widget Blueprint, Animation Blueprint, GAS Blueprint side

**Planned — new agents:**

- **Unreal AI Engineer** — Behavior Trees, EQS, NavMesh, Perception, Mass Entity framework
- **Unreal UI Engineer** — UMG, Common UI, MVVM data binding, Slate fundamentals, HUD architecture
- **Unreal Audio Engineer** — MetaSounds, Sound Classes, attenuation, audio modulation
- **Unreal Performance Profiler** — Unreal Insights, GPU/CPU profiling, memory budgeting, cook size audit
- **Unreal Cinematics Engineer** — Sequencer, cinematic cameras, nDisplay/virtual production
- **Unreal DevOps Engineer** — BuildGraph, Gauntlet, CI/CD, VCS large-file workflows

---

## Contributing

Contributions are welcome. The most useful contributions are:

- **New ref files** — if a UE5 pitfall isn't covered and would be caught by a rule, write a ref file and register it in the relevant agent
- **Rule corrections** — if a rule is wrong or outdated for a specific UE5 version, fix it with a note explaining why
- **Code examples** — ref files benefit from complete, copy-paste-ready examples with inline comments explaining the UE5 rationale
- **New agents** — if a domain isn't covered, a new agent file following the same structure is welcome

When editing: keep **agent files** focused on identity, workflow, and short topic summaries. Put technical depth in **ref files**. This keeps agent context small and ref files independently updatable.

See [`CLAUDE.md`](./CLAUDE.md) for the full authoring guidelines.

---

## Support This Project

If you find this useful, consider supporting its development.

[![Buy Me A Coffee](https://img.shields.io/badge/Buy_Me_A_Coffee-Support-FFDD00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/ashrafalnas)
[![Patreon](https://img.shields.io/badge/Patreon-Support-F96854?style=for-the-badge&logo=patreon&logoColor=white)](https://www.patreon.com/c/unrealpatr/)

| Platform            | Type                 | Link                                                                 |
| ------------------- | -------------------- | -------------------------------------------------------------------- |
| **Buy Me a Coffee** | One-time or monthly  | [buymeacoffee.com/ashrafalnas](https://buymeacoffee.com/ashrafalnas) |
| **Patreon**         | Recurring membership | [patreon.com/c/unrealpatr/](https://www.patreon.com/c/unrealpatr/)   |

---

## License

This project is free to use with no license restrictions.
