---
name: handover
description: Update dependency_graph.json (current_node, wip_status, edges), propose splits and folds, run the staleness check, and write the lossless WIP handover at config.handover_path. Use when pausing or ending work. Part of the graph-context-sync plugin.
---

# /graph-context-sync:handover

> Status: v3.0.0 (2026-09-24 — plugin packaging; protocol unchanged from v2)

Thin delegating command. Do not improvise its behaviour here.

1. Read `${CLAUDE_PLUGIN_ROOT}/skills/graph-context-sync/SKILL.md` in full.
2. Execute the section titled `/graph-context-sync:handover` exactly as written there. This command takes no arguments; ignore `$ARGUMENTS` if present.
3. All enforced rules R1–R8 of that file apply, including R4 (questions, proposals and checklists in
   `config.interaction_language`) and R6 (writes only inside the footprint).
