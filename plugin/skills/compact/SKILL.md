---
name: compact
description: Run the fold check over the whole dependency_graph.json and propose folding superseded decisions / resolved issues into their surviving decision. Part of the `graph` plugin.
---

# /graph:compact

> Status: v3.1.1 (2026-09-24 — wording fix for no-argument delegates, D24 content staleness, D25 generated handover tables; v3.1.0 = plugin `graph`; v3.0.0 = plugin packaging)

Thin delegating command. Do not improvise its behaviour here.

1. Read `${CLAUDE_PLUGIN_ROOT}/skills/protocol/SKILL.md` in full.
2. Execute the section titled `/graph:compact` exactly as written there. This command takes no arguments.
3. All enforced rules R1–R8 of that file apply, including R4 (questions, proposals and checklists in
   `config.interaction_language`) and R6 (writes only inside the footprint).
