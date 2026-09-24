---
name: hydrate
description: Load a node and its 1-hop/2-hop neighbourhood from dependency_graph.json, read every referenced file, and output the Impact Assessment Checklist. Required before modifying code. Usage: /graph-context-sync:hydrate <node_id>. Part of the graph-context-sync plugin.
arguments: [node_id]
---

# /graph-context-sync:hydrate

> Status: v3.0.0 (2026-09-24 — plugin packaging; protocol unchanged from v2)

Thin delegating command. Do not improvise its behaviour here.

1. Read `${CLAUDE_PLUGIN_ROOT}/skills/graph-context-sync/SKILL.md` in full.
2. Execute the section titled `/graph-context-sync:hydrate` exactly as written there. `$node_id` (the first argument) is the node to hydrate.
3. All enforced rules R1–R8 of that file apply, including R4 (questions, proposals and checklists in
   `config.interaction_language`) and R6 (writes only inside the footprint).
