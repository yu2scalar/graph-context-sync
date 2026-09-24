---
name: uninstall
description: Remove the graph-context-sync footprint from this project (dependency_graph.json, handover file, marked blocks) and verify CLAUDE.md and .gitignore are restored byte-identical; then tells you how to remove the plugin itself. Part of the graph-context-sync plugin.
disable-model-invocation: true
---

# /graph-context-sync:uninstall

> Status: v3.0.0 (2026-09-24 — plugin packaging; protocol unchanged from v2)

Thin delegating command. Do not improvise its behaviour here.

1. Read `${CLAUDE_PLUGIN_ROOT}/skills/graph-context-sync/SKILL.md` in full.
2. Execute the section titled `/graph-context-sync:uninstall` exactly as written there. This command takes no arguments; ignore `$ARGUMENTS` if present.
3. All enforced rules R1–R8 of that file apply, including R4 (questions, proposals and checklists in
   `config.interaction_language`) and R6 (writes only inside the footprint).
