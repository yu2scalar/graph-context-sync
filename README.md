# graph-context-sync

A [Claude Code](https://claude.com/claude-code) skill for graph-based project context management.

It exists to prevent two failure modes that become untrackable once a codebase is too large to "just
read the code": **duplicate or similar implementations** written because a session did not know an
existing component already covered the need, and **forgotten updates** where code changed but the design
document or decision governing it did not.

The skill keeps `dependency_graph.json`, a schema-validated **index** over the project (never a copy of its
content): components, design-document structure, decisions, issues, and the relations between them. It
forces the relevant 1-hop / 2-hop neighbourhood to be loaded before any code change and writes lossless
handovers when work pauses.

## Contents

```
.claude/skills/
├── graph-context-sync/
│   ├── SKILL.md                              # protocol: 6 sub-commands, 8 enforced rules, handover template
│   ├── schema/graph_schema.json              # JSON Schema (draft 2020-12) for dependency_graph.json
│   └── templates/graph_context.template.json # minimal valid seed graph
├── graph-install/SKILL.md                    # thin delegating commands so /graph-init etc.
├── graph-uninstall/SKILL.md                  # work as direct slash commands
├── graph-init/SKILL.md
├── graph-hydrate/SKILL.md
├── graph-handover/SKILL.md
└── graph-compact/SKILL.md
```

## Install into a project

Copy the skill directory, then let the skill do the rest with a bounded, reversible footprint:

```bash
mkdir -p .claude/skills
cp -r /path/to/graph-context-sync/.claude/skills/graph-* .claude/skills/
```

Then in Claude Code: `/graph-install` (appends one marked block to `CLAUDE.md` and `.gitignore`, creates the
seed graph, records checksums) followed by `/graph-init` (analyses the project, recommends settings, asks,
builds the graph). `/graph-uninstall` removes exactly that footprint and verifies the files are restored
byte-identical.

## Commands

| Command | What it does |
|---------|--------------|
| `/graph-install` | Create the seed graph, append marked blocks to `CLAUDE.md` / `.gitignore`, snapshot checksums. |
| `/graph-uninstall` | Remove the footprint, strip the marked blocks, verify byte-identical restoration. |
| `/graph-init [--reconfigure] [--reset-structure]` | Analyse the project, recommend and ask for `config` (design root, registries, handover path, language, components), derive the graph from design docs and registries. Idempotent; preserves human decisions unless `--reset-structure`. |
| `/graph-hydrate <node_id>` | Load the node plus 1-hop / 2-hop neighbours over every edge kind, read every referenced file, and emit the Impact Assessment Checklist: components, subgraph, inherited decision constraints, decisions to re-examine, existing capabilities, stale docs, blast radius. |
| `/graph-handover` | Update the graph, propose splits (growth) and folds (compaction), run the staleness check, write the handover with Active Task Pointer, Components + Subgraph, Hard Decisions Log, Unresolved Edges, Immediate Resume Trigger, Decision Drift, Staleness. |
| `/graph-compact` | Run the fold check on demand. |

## Graph model

**Hierarchy**: Root → `component` → `feature` → `function`. `decision` and `issue` nodes attach to the
structure. Root holds only `current_node`, `nodes`, `config`.

**Edges**: `part_of` (child → parent, at most one), `depends_on`, `affects`, `resolves`
(decision → issue), `supersedes` (decision → decision).

**Registry linkage**: decision / issue nodes carry `source_ref` (e.g. `D-022`, `TBD-24`); `config.registries`
says how such ids are recognised and which file holds their text. The graph indexes the registry, it does
not duplicate it.

**Growth**: a feature starts as one node. When attached decisions and issues reach `config.growth_threshold`
(default 5), or a decision applies to only part of it, the handover proposes splitting it into `function`
children; the parent becomes an index node.

**Compaction**: superseded decisions and resolved issues are folded into the surviving decision
(`folded: ["D-010", "TBD-03"]`) on approval, so the graph holds final decisions while the registry keeps history.

**Rebuild**: change `config` (for example the threshold) and re-run `/graph-init`; add `--reset-structure` to
re-propose splits and folds from scratch.

Validate a graph:

```bash
python3 -c "import json,jsonschema;jsonschema.Draft202012Validator(json.load(open('.claude/skills/graph-context-sync/schema/graph_schema.json'))).validate(json.load(open('dependency_graph.json')));print('OK')"
```

## Enforced rules

R1 cross-reference validation · R2 hydrate before modifying code · R3 handover fidelity ·
R4 interaction language (artifacts in English, questions in the user's language) · R5 structure follows
design docs · R6 bounded footprint · R7 components always visible · R8 no reinvention.
See `SKILL.md` for the full text.
