---
name: init
description: Create or refresh dependency_graph.json: analyse the project, recommend and ask for config (design root, registries, handover path, language, components), derive the graph from design docs and registries. Idempotent. Flags: --reconfigure, --reset-structure. Part of the `graph` plugin.
arguments: [flags]
---

# /graph:init

> Status: v3.1.0 (2026-09-24 — plugin renamed `graph`, protocol skill renamed `protocol`; v3.0.0 = plugin packaging)

Thin delegating command. Do not improvise its behaviour here.

1. Read `${CLAUDE_PLUGIN_ROOT}/skills/protocol/SKILL.md` in full.
2. Execute the section titled `/graph:init` exactly as written there. `$ARGUMENTS` may contain `--reconfigure` and/or `--reset-structure`.
3. All enforced rules R1–R8 of that file apply, including R4 (questions, proposals and checklists in
   `config.interaction_language`) and R6 (writes only inside the footprint).
