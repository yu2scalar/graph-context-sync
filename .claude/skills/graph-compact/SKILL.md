---
name: graph-compact
description: Run the fold check over the whole dependency_graph.json and propose folding superseded decisions / resolved issues into their surviving decision. Delegates to the graph-context-sync protocol.
---

# /graph-compact

This is a thin delegating command. Do not improvise its behaviour here.

1. Read `.claude/skills/graph-context-sync/SKILL.md` in full.
2. Execute the section titled `/graph-compact` exactly as written there, with the arguments given to this
   command (for `/graph-hydrate`, the first argument is `<node_id>`; for `/graph-init`, flags
   `--reconfigure` / `--reset-structure` may be present).
3. All enforced rules R1–R8 of that file apply, including R4 (questions, proposals and checklists in
   `config.interaction_language`) and R6 (writes only inside the footprint).

Equivalent invocation: `/graph-context-sync compact <args>`.
