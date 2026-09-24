---
name: install
description: Install graph-context-sync into this project with a bounded, reversible footprint: seed dependency_graph.json, append marked blocks to CLAUDE.md and .gitignore, snapshot sha256. Nothing else is written. Use when adding the graph to a project for the first time. Part of the `graph` plugin.
---

# /graph:install

> Status: v3.1.0 (2026-09-24 — plugin renamed `graph`, protocol skill renamed `protocol`; v3.0.0 = plugin packaging)

Thin delegating command. Do not improvise its behaviour here.

1. Read `${CLAUDE_PLUGIN_ROOT}/skills/protocol/SKILL.md` in full.
2. Execute the section titled `/graph:install` exactly as written there. This command takes no arguments; ignore `$ARGUMENTS` if present.
3. All enforced rules R1–R8 of that file apply, including R4 (questions, proposals and checklists in
   `config.interaction_language`) and R6 (writes only inside the footprint).
