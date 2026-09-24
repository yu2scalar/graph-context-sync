---
name: init
description: Create or refresh dependency_graph.json: analyse the project, recommend and ask for config (design root, registries, handover path, language, components), derive the graph from design docs and registries. Idempotent. Flags: --reconfigure, --reset-structure. Part of the graph-context-sync plugin.
arguments: [flags]
---

# /graph-context-sync:init

> Status: v3.0.0 (2026-09-24 — plugin packaging; protocol unchanged from v2)

Thin delegating command. Do not improvise its behaviour here.

1. Read `${CLAUDE_PLUGIN_ROOT}/skills/graph-context-sync/SKILL.md` in full.
2. Execute the section titled `/graph-context-sync:init` exactly as written there. `$ARGUMENTS` may contain `--reconfigure` and/or `--reset-structure`.
3. All enforced rules R1–R8 of that file apply, including R4 (questions, proposals and checklists in
   `config.interaction_language`) and R6 (writes only inside the footprint).
