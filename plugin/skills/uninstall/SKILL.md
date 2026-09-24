---
name: uninstall
description: Remove the graph-context-sync footprint from this project (dependency_graph.json, handover file, marked blocks) and verify CLAUDE.md and .gitignore are restored byte-identical; then tells you how to remove the plugin itself. Part of the `graph` plugin.
disable-model-invocation: true
---

# /graph:uninstall

> Status: v3.1.0 (2026-09-24 — plugin renamed `graph`, protocol skill renamed `protocol`; v3.0.0 = plugin packaging)

Thin delegating command. Do not improvise its behaviour here.

1. Read `${CLAUDE_PLUGIN_ROOT}/skills/protocol/SKILL.md` in full.
2. Execute the section titled `/graph:uninstall` exactly as written there. This command takes no arguments; ignore `$ARGUMENTS` if present.
3. All enforced rules R1–R8 of that file apply, including R4 (questions, proposals and checklists in
   `config.interaction_language`) and R6 (writes only inside the footprint).
