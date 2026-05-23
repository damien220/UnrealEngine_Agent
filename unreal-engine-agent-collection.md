---
title: "Unreal Engine Agent Collection"
description: "Specialized Claude Code sub-agents encoding AAA-grade Unreal Engine 5 conventions so developers never re-explain context."
type: "Library"
tags: ["Claude Code", "Unreal Engine 5", "AI Agents", "Game Development", "Markdown"]
featured: false
image: "/images/unreal-engine-agent-collection.png"
date: "2026-05"
---

## Overview

A collection of specialist Claude Code sub-agent definitions for Unreal Engine 5 development. Each agent encodes deep UE5 domain knowledge — GAS setup, actor replication, World Partition, Blueprint conventions — so developers get correct, convention-aware answers without re-establishing ground rules every session. Agents load topic-specific reference files on demand, keeping per-session context minimal while providing full technical depth when needed.

## Key Features

- Four specialist agents covering C++/Blueprint architecture, multiplayer networking, open-world level design, and Blueprint authoring
- 29 reference files across agent domains, each scoped to a single UE5 topic and loaded only when relevant
- Structured agent format (YAML frontmatter + fixed markdown sections) compatible with the Claude Code FleetView harness
- Quantitative success metrics and copy-paste-ready C++ code blocks in every agent
- Roadmap of six additional planned agents (AI, UI, Audio, Performance, Cinematics, DevOps)

## Technical Decisions

The key architectural decision is separating agent identity (personality, workflow, short summaries) from knowledge (ref files). This means updating a UE5 rule — say, a new Nanite instance ceiling or a GAS API change — requires editing only the relevant ref file, never the agent's core definition. The on-demand ref loading pattern keeps Claude Code's per-invocation context window small, which directly improves response quality and reduces cost on long sessions.
