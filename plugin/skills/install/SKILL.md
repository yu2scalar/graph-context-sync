---
name: install
description: Install graph-context-sync into this project with a bounded, reversible footprint: seed dependency_graph.json, append marked blocks to CLAUDE.md and .gitignore, snapshot sha256. Nothing else is written. Use when adding the graph to a project for the first time. Part of the graph-context-sync plugin.
---

# /graph-context-sync:install

> Status: v3.0.0 (2026-09-24 — plugin packaging; protocol unchanged from v2)

Thin delegating command. Do not improvise its behaviour here.

1. Read `${CLAUDE_PLUGIN_ROOT}/skills/graph-context-sync/SKILL.md` in full.
2. Execute the section titled `/graph-context-sync:install` exactly as written there. This command takes no arguments; ignore `$ARGUMENTS` if present.
3. All enforced rules R1–R8 of that file apply, including R4 (questions, proposals and checklists in
   `config.interaction_language`) and R6 (writes only inside the footprint).
