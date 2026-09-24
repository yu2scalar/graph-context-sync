---
name: hydrate
description: Load a node and its 1-hop/2-hop neighbourhood from dependency_graph.json, read every referenced file, and output the Impact Assessment Checklist. Required before modifying code. Usage: /graph:hydrate <node_id>. Part of the `graph` plugin.
arguments: [node_id]
---

# /graph:hydrate

> Status: v3.1.1 (2026-09-24 — wording fix for no-argument delegates, D24 content staleness, D25 generated handover tables; v3.1.0 = plugin `graph`; v3.0.0 = plugin packaging)

Thin delegating command. Do not improvise its behaviour here.

1. Read `${CLAUDE_PLUGIN_ROOT}/skills/protocol/SKILL.md` in full.
2. Execute the section titled `/graph:hydrate` exactly as written there. `$node_id` (the first argument) is the node to hydrate.
3. All enforced rules R1–R8 of that file apply, including R4 (questions, proposals and checklists in
   `config.interaction_language`) and R6 (writes only inside the footprint).
