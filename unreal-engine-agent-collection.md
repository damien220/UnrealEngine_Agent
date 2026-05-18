---
title: Unreal Engine Agent Collection
description: Specialized Claude Code sub-agents for UE5 development, encoding AAA-grade Unreal Engine conventions as persistent, on-demand knowledge.
type: CLI Tool
tags:
  - Claude Code
  - Unreal Engine 5
  - AI Agents
  - C++
  - Game Development
featured: false
image: /images/unreal-engine-agent-collection.png
date: 2026-05
---

## Overview

Unreal Engine Agent Collection is a set of specialized Claude Code sub-agent definition files for Unreal Engine 5 development. Working with a general-purpose AI on UE5 means re-explaining the same rules every session — that `UPROPERTY()` is mandatory, that Blueprint Tick is a performance liability, that GAS needs three specific module dependencies. These agents encode that knowledge permanently, so every session starts with AAA-grade UE5 conventions already in place. Three specialists are included: a Systems Engineer, a Multiplayer Architect, and a World Builder, each with a focused domain and its own library of on-demand reference files.

## Key Features

- **Three specialist agents** — Systems Engineer (C++/Blueprint, GAS, Nanite/Lumen), Multiplayer Architect (replication, server authority, dedicated server), and World Builder (World Partition, Landscape, PCG, HLOD)
- **On-demand reference architecture** — 12 topic-scoped ref files loaded only when the task requires them, keeping per-session context minimal while providing full technical depth
- **Separation of behavior and knowledge** — agent files define identity and workflow; ref files define rules and code examples, independently updatable without touching the agent
- **AAA-rule enforcement** — critical UE5 rules encoded as hard constraints: `IsValid()` over `!= nullptr`, no Blueprint Tick, UPROPERTY on all UObject pointers, GAS replication through ASC only
- **Copy-paste-ready code** — all ref files include complete C++ examples with inline comments explaining the UE5 rationale, not just what the code does
- **Extensible by design** — new rules, topics, or agents follow a documented pattern: narrow rules go in the agent file, anything requiring examples gets its own ref file

## Technical Decisions

The key architectural decision was separating agent identity (the `.md` root files loaded by FleetView) from domain knowledge (the `refs/` subtree). This allows ref files to be updated for new UE5 versions, corrected, or expanded without destabilizing the agent's behavior or workflow. The on-demand loading trigger — a 2–3 sentence summary plus a `Read refs/... when:` condition — acts as a relevance gate that keeps context windows small on simple tasks and deep on complex ones. Markdown was chosen as the sole format because it is both human-readable for authoring and directly consumable by the Claude Code harness without any build step.
