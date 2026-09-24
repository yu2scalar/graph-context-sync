# graph-context-sync

A [Claude Code](https://claude.com/claude-code) skill for graph-based project context
management. It keeps an explicit, schema-validated graph of features, decisions, tasks and
issues (`dependency_graph.json`), forces the relevant 1-hop / 2-hop neighbourhood to be loaded
before any code change, and writes a lossless handover document when work pauses.

## Contents

```
.claude/skills/graph-context-sync/
├── SKILL.md                              # protocol: commands, enforced rules, handover template
├── schema/graph_schema.json              # JSON Schema (draft 2020-12) for dependency_graph.json
└── templates/graph_context.template.json # minimal valid seed graph
```

## Install into a project

Copy the skill directory into the target project:

```bash
mkdir -p .claude/skills
cp -r /path/to/graph-context-sync/.claude/skills/graph-context-sync .claude/skills/
```

Optionally add to the project's `CLAUDE.md`:

```markdown
## CRITICAL PROTOCOL
- You must strictly follow the skill rules defined in `.claude/skills/graph-context-sync/SKILL.md`.
- Before modifying any feature or fixing bugs, verify if `dependency_graph.json` exists. If so, invoke the logic of `/graph-hydrate <node_id>` to load 1-hop/2-hop dependencies first.
- When ending a session or pausing work, invoke `/graph-handover` to generate `.context/WIP_HANDOVER.md`. Never write lossy, generic summaries.
```

## Commands

| Command | What it does |
|---------|--------------|
| `/graph-init` | Scans `docs/**/*.md` and the code files those docs reference, then creates or refreshes `dependency_graph.json`. |
| `/graph-hydrate <node_id>` | Loads the node plus its 1-hop and 2-hop neighbours (forward and reverse `depends_on` / `affects`), reads every referenced `docs` and `code_targets` file, and emits an Impact Assessment Checklist. Code may not be modified until every check is ticked. |
| `/graph-handover` | Updates `current_node`, `wip_status` and edges, then writes `.context/WIP_HANDOVER.md` with Active Task Pointer, Subgraph Context Range, Hard Decisions Log, Unresolved Edges and Immediate Resume Trigger. Decisions and identifiers are preserved verbatim. |

## Graph format

Each node requires `id`, `type` (`feature` | `decision` | `task` | `issue`), `name`, `docs[]`
and `code_targets[]`, and may carry `depends_on[]`, `affects[]` and `wip_status`
(`DONE` | `IN_PROGRESS` | `BLOCKED`). Node ids match `^[a-z0-9][a-z0-9_-]*$`, the key in
`nodes` must equal the node's `id`, and every edge target must exist.

Validate a graph:

```bash
python3 -c "import json,jsonschema;jsonschema.Draft202012Validator(json.load(open('.claude/skills/graph-context-sync/schema/graph_schema.json'))).validate(json.load(open('dependency_graph.json')));print('OK')"
```
