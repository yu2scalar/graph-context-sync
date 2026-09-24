---
name: graph-context-sync
description: Graph-based project context management. Use for /graph-init (build or refresh dependency_graph.json from docs/), /graph-hydrate <node_id> (load 1-hop and 2-hop dependencies and produce an Impact Assessment Checklist before touching code), and /graph-handover (update the graph and write .context/WIP_HANDOVER.md without lossy summarization). Also triggers on "dependency graph", "hydrate node", "handover", "WIP handover", "context graph".
---

# graph-context-sync

Protocol for keeping an explicit, machine-checkable graph of project context and for
loading the right slice of that graph before any code change.

## Files

| Path | Role |
|------|------|
| `dependency_graph.json` (project root) | The graph. Must validate against the schema below. |
| `.claude/skills/graph-context-sync/schema/graph_schema.json` | JSON Schema (draft 2020-12) for the graph. |
| `.claude/skills/graph-context-sync/templates/graph_context.template.json` | Minimal valid seed graph (`current_node: null`, `nodes: {}`). |
| `.context/WIP_HANDOVER.md` | Handover document written by `/graph-handover`. |

Node shape (see schema for the authoritative definition):

- required: `id`, `type` (`feature` | `decision` | `task` | `issue`), `name`, `docs[]`, `code_targets[]`
- optional: `depends_on[]`, `affects[]`, `wip_status` (`DONE` | `IN_PROGRESS` | `BLOCKED`)
- ids match `^[a-z0-9][a-z0-9_-]*$`

## Command recognition

The three commands below are sub-commands of this skill. Treat any of the following as an
invocation: `/graph-init`, `/graph-hydrate <node_id>`, `/graph-handover`, or
`/graph-context-sync <init|hydrate <node_id>|handover>`. Natural-language requests such as
"rebuild the dependency graph", "hydrate the auth node", or "write the handover" map to the
same commands.

---

## `/graph-init`

Purpose: generate `dependency_graph.json` if absent, or refresh it if present, so that it
matches `graph_schema.json`.

### Procedure

1. **Load existing graph.** If `dependency_graph.json` exists, read it and validate it
   (see Enforced Rules). Keep every existing node, edge, `wip_status`, and `current_node`
   unless a scanned source contradicts it. Never drop a node just because a scan did not
   re-discover it; mark it in the report instead.
   If absent, start from `templates/graph_context.template.json`.
2. **Scan `docs/`.** Read every `docs/**/*.md`. For each document, identify candidate
   nodes:
   - plan / design / spec documents → `feature` or `task`
   - ADRs, "Decision" sections, decision tables → `decision`
   - bug reports, known-issue lists, TODO/FIXME sections → `issue`
   Record the document path in the node's `docs`.
3. **Follow references from docs into code.** Collect every project-relative file path
   mentioned in the scanned docs (code blocks, inline code, tables, links). Read each file
   that exists and assign it to the `code_targets` of the node(s) whose doc referenced it.
   Do **not** walk the source tree blindly. A file that no doc mentions is out of scope for
   init; it can be added later by `/graph-handover`.
4. **Derive edges.**
   - `depends_on`: explicit "depends on", "requires", "after", "blocked by", "based on"
     phrasing, or a doc that cites another node's doc as a prerequisite.
   - `affects`: explicit "affects", "impacts", "changes", "breaks" phrasing, or shared
     `code_targets` between two nodes (the node that owns the file is affected by the node
     that modifies it).
   Every edge target must be a node id that exists in `nodes`. If the target does not exist
   yet, create it as a stub node (`type` inferred, `docs`/`code_targets` may be empty) rather
   than emitting a dangling edge.
5. **Assign ids.** kebab-case or snake_case, lowercase, derived from the document or feature
   name. The object key in `nodes` must be identical to `node.id`.
6. **Validate and write.** Run the Enforced Rules checks. Write `dependency_graph.json` with
   2-space indentation and keys in the order: `$schema` (optional), `current_node`, `nodes`.
7. **Report.** Output a table of nodes added / updated / unchanged / not-rediscovered, and
   the list of edges added. Do not silently change `current_node`; if the existing graph has
   one, keep it.

---

## `/graph-hydrate <node_id>`

Purpose: load the full 1-hop and 2-hop neighbourhood of `<node_id>` into working context and
produce an Impact Assessment Checklist **before any code modification**.

### Procedure

1. **Resolve the node.** Read `dependency_graph.json`. If `<node_id>` is not a key of
   `nodes`, stop and list the closest matching ids; do not guess.
2. **Compute the subgraph.**
   - hop 0: `<node_id>`
   - hop 1: every id in the node's `depends_on` and `affects`, plus every node whose
     `depends_on` or `affects` contains `<node_id>` (reverse edges)
   - hop 2: apply the same expansion to every hop-1 node
   Deduplicate. Record the hop distance of each node.
3. **Read every referenced file.** For each node in hops 0–2, read every path in `docs` and
   `code_targets` with the file reading tool. Do not skim or sample; if a file is very large,
   read it in ranges until covered. Note any path that does not exist on disk.
4. **Set `current_node`.** Update `current_node` in `dependency_graph.json` to `<node_id>`
   and, if its `wip_status` is absent or `DONE`, leave `wip_status` untouched until the user
   confirms work has started.
5. **Output the Impact Assessment Checklist** in exactly this shape:

```markdown
## Impact Assessment Checklist — <node_id>

### Subgraph
| hop | id | type | wip_status | edge from origin |
|-----|----|------|------------|------------------|
| 0 | <node_id> | ... | ... | — |
| 1 | ... | ... | ... | depends_on / affects / reverse-depends_on / reverse-affects |
| 2 | ... | ... | ... | via <hop-1 id> |

### Files loaded
| node | kind | path | status |
|------|------|------|--------|
| ... | docs / code_targets | ... | read / MISSING |

### Constraints inherited from decisions
- <one bullet per `decision` node in the subgraph: the decision, verbatim identifiers it fixes>

### Blast radius
- Files in hop-0 `code_targets` that also appear in another node's `code_targets`: ...
- Nodes with `wip_status: IN_PROGRESS` or `BLOCKED` in the subgraph: ...

### Pre-modification checks
- [ ] All hop-1 and hop-2 files read (no MISSING rows, or each MISSING row acknowledged)
- [ ] No conflicting IN_PROGRESS work on shared code_targets
- [ ] Decision constraints above will be honoured
- [ ] `current_node` set to <node_id>
```

Only after every box can be ticked may code modification begin.

---

## `/graph-handover`

Purpose: persist the exact state of work so a fresh session can resume with zero
re-discovery.

### Procedure

1. **Update the graph.**
   - `current_node`: the node being worked on, or `null` if work is fully complete.
   - `wip_status` of the current node and any node touched this session: `DONE`,
     `IN_PROGRESS`, or `BLOCKED`.
   - Edges: add any `depends_on` / `affects` discovered during the work. Add new nodes for
     newly discovered decisions or issues. Add newly touched files to `code_targets` and newly
     written docs to `docs`.
   - Validate against the Enforced Rules, then write.
2. **Write `.context/WIP_HANDOVER.md`** using the Handover Template below. Create the
   `.context/` directory if needed. Overwrite the previous handover; the graph is the durable
   history, the handover is the live pointer.
3. **Fidelity rule.** The handover must preserve, verbatim:
   - every explicit technical decision made this session and its stated reason
   - every identifier that was chosen or renamed: variable, function, class, file, config key,
     schema field, enum value, CLI flag
   - every edge that was discussed but not resolved (an open dependency, an unconfirmed
     impact, a target that may need changing)
   Do not compress these into phrases such as "refactored the auth module" or "various fixes".
   If a decision was made, name the option chosen and the options rejected.

### Handover Template

```markdown
# WIP HANDOVER — <project name>
Generated: <YYYY-MM-DD HH:MM> · Graph: `dependency_graph.json` · Schema: `.claude/skills/graph-context-sync/schema/graph_schema.json`

## 1. Active Task Pointer
- current_node: `<node_id>` (`<type>`, wip_status: `<status>`)
- Name: <node.name>
- Goal of this node in one sentence, as originally stated by the user: "<verbatim>"
- Work completed this session (concrete, file-level):
  - `<path>`: <what changed>
- Work NOT yet done (concrete):
  - <item>

## 2. Subgraph Context Range
Nodes that were hydrated and must be re-hydrated on resume (`/graph-hydrate <current_node>` reproduces this).
| hop | id | type | wip_status | why it matters to the active task |
|-----|----|------|------------|-----------------------------------|
| 0 | ... | ... | ... | ... |
| 1 | ... | ... | ... | ... |
| 2 | ... | ... | ... | ... |

Files read this session that are outside the subgraph's `docs` / `code_targets` (candidates to add to the graph):
- `<path>` — <reason it was needed>

## 3. Hard Decisions Log
One entry per explicit technical decision. Verbatim identifiers. No paraphrase.
| # | Decision | Chosen | Rejected alternatives | Reason (as stated) | Fixed identifiers |
|---|----------|--------|-----------------------|--------------------|-------------------|
| D1 | ... | ... | ... | ... | `var_name`, `ClassName`, `config.key`, ... |

## 4. Unresolved Edges
Dependencies or impacts that were raised but not confirmed, closed, or reflected in code.
| # | From node | To node / file | Kind (depends_on / affects / unknown) | What is unresolved | Who or what resolves it |
|---|-----------|----------------|----------------------------------------|--------------------|-------------------------|
| U1 | ... | ... | ... | ... | ... |

## 5. Immediate Resume Trigger
The exact first actions for the next session, in order. No discovery step should be needed.
1. Run `/graph-hydrate <current_node>` and confirm the Impact Assessment Checklist matches section 2.
2. Open `<path>` at `<symbol or line>`; the next edit is: <precise description>.
3. Run: `<command>` and expect: <expected output>.
4. Blocking question for the user, if any: "<verbatim question>"
```

---

## Enforced Rules

These rules are mandatory for every command and for any manual edit of the graph.

### R1 — Cross-Reference Validation
1. **Key equals id.** For every entry `nodes[k]`, `nodes[k].id === k`. A mismatch is an
   error: fix the key (never the id, since other edges may reference it) and report the fix.
2. **Edge targets exist.** Every value in any `depends_on` or `affects` array, and
   `current_node` when non-null, must be a key of `nodes`. A dangling reference is an error:
   either create the missing node or remove the edge, and report which was done.
3. **Schema validity.** The document must validate against `graph_schema.json`
   (draft 2020-12). When a validator is available, run it, for example:

   ```bash
   python3 -c "import json,jsonschema;jsonschema.Draft202012Validator(json.load(open('.claude/skills/graph-context-sync/schema/graph_schema.json'))).validate(json.load(open('dependency_graph.json')));print('OK')"
   ```

   When no validator is available, check the required fields, enums, and id pattern manually
   and say so in the report.
4. **No self-edges.** A node may not list itself in `depends_on` or `affects`.

### R2 — Hydration Rule
- **Never modify code before hydrating.** If `dependency_graph.json` exists, any edit to a
  file under some node's `code_targets`, and any feature change or bug fix in general, must be
  preceded by `/graph-hydrate <node_id>` for the relevant node, with every item of the
  Pre-modification checks ticked.
- If the relevant node does not exist yet, create it first (via `/graph-init` refresh or a
  manual addition that passes R1), then hydrate.
- If the user explicitly asks to skip hydration, state the risk in one sentence, record the
  skip in the next handover's Unresolved Edges, and proceed.

### R3 — Handover Fidelity
- `/graph-handover` output must satisfy the Fidelity rule above. A handover containing only
  generic summaries is non-compliant and must be rewritten before the session ends.
- Sections 1–5 of the template are all mandatory. An empty section is written as
  `- none` so that absence is explicit, not accidental.
