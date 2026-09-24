---
name: graph-context-sync
description: Graph-based project context management (v2). Keeps dependency_graph.json as a schema-validated index over a project's components, design documents and decision/issue registries, forces 1-hop/2-hop hydration before code changes, and writes lossless handovers. Sub-commands: /graph-install, /graph-uninstall, /graph-init, /graph-hydrate <node_id>, /graph-handover, /graph-compact. Also triggers on "dependency graph", "hydrate node", "handover", "WIP handover", "context graph", "graph init", "compact graph".
---

# graph-context-sync (v2)

## Purpose

Two failure modes this skill exists to prevent, both of which become untrackable once a codebase is
large enough that "just read the code" stops working:

1. **Duplicate or similar implementations**, typically because a session did not know an existing
   component or function already covered the need.
2. **Forgotten updates**: code changed, the design document or decision that governs it did not.

It does so by keeping an explicit, machine-checkable **index graph** over the project. The graph never
holds content; it holds references to where the content lives and the relations between them.

## Files

| Path | Role |
|------|------|
| `dependency_graph.json` (project root) | The graph. Must validate against the schema. |
| `.claude/skills/graph-context-sync/schema/graph_schema.json` | JSON Schema, draft 2020-12. Authoritative for shapes. |
| `.claude/skills/graph-context-sync/templates/graph_context.template.json` | Minimal valid seed graph. |
| `.claude/skills/graph-{install,uninstall,init,hydrate,handover,compact}/SKILL.md` | Six thin delegating skills so that `/graph-init` etc. work as direct slash commands (D16). Each reads this file and executes the matching section. |
| `config.handover_path` (default `.context/WIP_HANDOVER.md`) | Handover written by `/graph-handover`. |

## Data model (summary; schema is authoritative)

**Root** is closed: `current_node`, `nodes`, `config` only.

**Hierarchy**: Root → `component` → `feature` → `function`. `decision` and `issue` nodes attach to a
component, feature or function. `task` exists for manual use and is never generated.

**Node fields**: required `id`, `type`, `name`, `docs[]`, `code_targets[]`. Optional edges:
`part_of` (≤1, child → parent), `depends_on`, `affects`, `resolves` (decision → issue only),
`supersedes` (decision → decision only). Optional `source_ref` (registry id, decision/issue only,
single-valued), `folded[]` (registry ids absorbed by compaction, decision only), `wip_status`.

**Edge direction conventions**

| Relation | Edge |
|----------|------|
| feature belongs to component / function belongs to feature | child `part_of` parent |
| feature needs another feature | `depends_on` |
| feature/function expects something from another component | `depends_on` → that component's function/feature node |
| issue impacts a feature/function/component | issue `affects` target |
| decision constrains a feature/function/component | decision `affects` target |
| decision resolves an issue | decision `resolves` issue |
| decision replaces or refines an earlier decision | decision `supersedes` earlier decision |
| issue was raised by a decision | issue `depends_on` decision |

**`config` keys**

| Key | Default | Meaning |
|-----|---------|---------|
| `interaction_language` | inferred | language for every question, recommendation, approval, checklist shown to the user |
| `handover_path` | `.context/WIP_HANDOVER.md` | where `/graph-handover` writes |
| `design_root` | detected | directory whose document structure the feature/function layer mirrors |
| `docs_scope` | `<design_root>/**/*.md` | globs `/graph-init` reads |
| `registries[]` | `[]` | `{type, id_pattern, file}`: how decision/issue ids are recognised and where their text lives |
| `growth_threshold` | 5 | attached decision+issue count at which a split is proposed |
| `install` | set by install | sha256 snapshots for uninstall verification |

Not in config, by decision: `code_roots` (derived: union of component nodes' `code_targets`),
`project_name`, `version` (git / CLAUDE.md own them).

## Command recognition

`/graph-install`, `/graph-uninstall`, `/graph-init`, `/graph-hydrate <node_id>`, `/graph-handover` and
`/graph-compact` are registered as their own slash commands via the six delegating skills (D16; this
superseded S2, under which only `/graph-context-sync <sub>` existed). The long form
`/graph-context-sync <install|uninstall|init|hydrate <node_id>|handover|compact>` still works. Natural-language
equivalents ("rebuild the dependency graph", "hydrate the commit-protocol node", "write the handover",
"compact the decisions") map to the same commands.

---

## `/graph-install`

Purpose: put the skill into a host project with a bounded, reversible footprint (rule R6).

1. If invoked from the upstream repo, copy `.claude/skills/graph-context-sync/` **and the six delegating
   skill directories** `.claude/skills/graph-{install,uninstall,init,hydrate,handover,compact}/` into the target.
2. Record sha256 of the target's `CLAUDE.md` and `.gitignore` as they are now (null if absent).
3. Create `dependency_graph.json` from the template if absent.
4. Append to `CLAUDE.md` (create if absent) exactly one marked block:
   ```markdown
   <!-- graph-context-sync:begin -->
   ## CRITICAL PROTOCOL (graph-context-sync)
   - You must strictly follow the skill rules defined in `.claude/skills/graph-context-sync/SKILL.md`.
   - Before modifying any feature or fixing bugs, verify if `dependency_graph.json` exists. If so, invoke the logic of `/graph-hydrate <node_id>` to load 1-hop/2-hop dependencies first.
   - When ending a session or pausing work, invoke `/graph-handover` to generate the handover at `config.handover_path`. Never write lossy, generic summaries.
   - Registry mapping (decision / issue ids → files) = `dependency_graph.json` → `config.registries`.
   <!-- graph-context-sync:end -->
   ```
5. Append to `.gitignore` (create if absent) exactly one marked block:
   ```
   # graph-context-sync:begin
   .context/
   # graph-context-sync:end
   ```
   Omit the `.context/` line if `config.handover_path` will live in a tracked directory; keep the markers.
6. Write `config.install` `{installed_at, skill_version, claude_md_sha256_before, gitignore_sha256_before}`.
7. Print the footprint (every path created or modified). Ask, in the interaction language, before writing anything.

## `/graph-uninstall`

1. Compute the footprint: the skill directory, the six delegating skill directories, `dependency_graph.json`, the file at `config.handover_path`
   (and `.context/` if that is its directory and it is otherwise empty), the marked block in `CLAUDE.md`,
   the marked block in `.gitignore`.
2. Show the list with a per-path action (delete file / strip block / delete empty dir) and whether the path
   is git-tracked. Ask for approval.
3. On approval: delete files and dirs the skill created; strip exactly the text between and including the
   markers (plus one trailing newline) from `CLAUDE.md` and `.gitignore`; if a file becomes empty and the
   skill created it, delete it.
4. Verify: sha256 of `CLAUDE.md` and `.gitignore` now equal `config.install.*_before` (null = file should
   not exist). Report "restored byte-identical" or list differences (which can only come from user edits
   outside the markers; those are kept).
5. Warn once if any removed path was git-tracked so the user can `git rm` in the same commit.

Nothing outside the footprint is ever touched. Anything Claude stored in its own memory cannot be
uninstalled, so the skill never writes memory (R6).

---

## `/graph-init [--reconfigure] [--reset-structure]`

Purpose: create or refresh `dependency_graph.json`. Idempotent (F8).

### Step 0 — project analysis and configuration Q&A
Run when `config` is incomplete or `--reconfigure` is given.

1. Detect and tabulate, then **recommend and ask** (interaction language):
   - `design_root`: candidates `docs/design/`, `design/`, `docs/`, `doc/`; prefer the one with a README or
     index and the most cross-references.
   - `docs_scope`: default `<design_root>/**/*.md`; offer to add plan / handover docs if found.
   - `registries`: files matching `decision-log*`, `adr*`, `decisions*` → type decision; `tbd*`, `issues*`,
     `open-questions*` → type issue. Derive `id_pattern` from ids actually present (e.g. `^D-\d{3}$`,
     `^TBD-\d{2}$`) and show three sample ids per registry as evidence.
   - `handover_path`: if the repo already commits handover documents under a docs folder, recommend
     `<that folder>/WIP_HANDOVER.md` (tracked); otherwise `.context/WIP_HANDOVER.md` (ignored).
   - `interaction_language`: infer from `CLAUDE.md` and recent user messages; confirm.
   - `growth_threshold`: default 5; state that it can be changed later and the graph rebuilt.
   - **components**: top-level directories that contain build files or sources, excluding `build/`,
     `.gradle/`, `.idea/`, `node_modules/`, `target/`, `dist/`, `.git/`, docs folders. Propose one
     `component` node each with `code_targets` = that directory; the primary source tree (e.g. `src/`)
     becomes `core` unless the user names it otherwise. The user confirms names, paths, and may add or
     remove components. Proposing a component is always a user decision (R8).
     Scope note (D17): component detection is a *listing of top-level directory names* only. It never
     reads source files. The "no blind source walk" rule (S3) governs how `code_targets` are derived in
     Step 2 (from paths referenced by design documents), not this listing.
2. Persist answers to `config` and create the component nodes.

### Step 1 — load and preserve
If the graph exists, validate it (R1). Preserve `current_node`, `wip_status`, `part_of`, `folded`, `config`
unless `--reset-structure` (which discards `part_of` and `folded` and re-proposes splits/folds under the
current `growth_threshold`). Never drop a node because a scan did not rediscover it; report it instead.

### Step 2 — derive structure from design docs (R5)
For each document in `docs_scope`:
- One `feature` node per design document, `part_of` the component whose `code_targets` its referenced
  code falls under (ask if ambiguous; an index/overview document becomes `docs` of the component instead
  of a feature).
- A document with clearly separate top-level sections may yield `function` children; do **not** split on
  first init unless the structure is explicit. Splitting is normally a growth proposal at handover.
- Collect project-relative code paths mentioned in the document (code spans, tables, links) into
  `code_targets`; each must fall under some component's roots, otherwise report it as unplaced.
- Never generate `task` nodes.

### Step 3 — decisions and issues (R5, OP1 = C)
Scan the documents in scope and the registry files for ids matching `config.registries[].id_pattern`.
Create a `decision` / `issue` node **only** when the id is referenced from a document in scope or from the
registry text of another referenced id. Set `source_ref` verbatim; `docs` = the registry file; `part_of`
= the feature/function/component whose document referenced it (component when cross-cutting).

### Step 4 — edges
- `affects`: decision/issue → the nodes whose documents reference it.
- `resolves`: from registry text such as "resolves TBD-24", "closes", "決定により解消", or a TBD entry that
  names the D-id that closed it.
- `supersedes`: from registry text such as "supersedes D-010", "replaces", "上書き", "置き換え".
- `depends_on`: from explicit "depends on / requires / after / blocked by / 前提" phrasing; for
  cross-component expectations, target the providing component's function/feature; create a stub node under
  the **providing** component if it does not exist (never under the requesting one).
- Every edge target must exist; create stubs rather than dangling edges.

### Step 5 — validate, diff, write, report
Run R1. Show a before/after diff (nodes added / updated / removed / merged; edges added) in the interaction
language and ask before writing. Write with 2-space indentation, key order `$schema`, `current_node`,
`nodes`, `config`. Report nodes not rediscovered and code paths not placed under any component.

---

## `/graph-hydrate <node_id>`

Purpose: load the full 1-hop / 2-hop neighbourhood and produce the Impact Assessment Checklist
**before any code modification** (R2).

1. Resolve `<node_id>`; if absent, list the closest ids and stop. Do not guess.
2. Subgraph: hop 0 = the node; hop 1 = every target and every source of any edge kind
   (`part_of` both directions, `depends_on`, `affects`, `resolves`, `supersedes`); hop 2 = same expansion
   from hop 1. Record hop distance and the edge path.
3. Read every `docs` and `code_targets` path of every node in hops 0–2 in full (ranges for large files).
   For `decision`/`issue` nodes, read the `source_ref` entry in the registry file. Note missing paths.
4. Set `current_node` = `<node_id>`.
5. Staleness (F10): for each node in the subgraph, compare the newest git commit touching any
   `code_targets` with the newest touching any `docs` (registry file for decision/issue). Fall back to mtime
   without git. Flag code-newer-than-docs, missing paths, and `source_ref` not found in the registry file.
6. Output the checklist in exactly this shape:

```markdown
## Impact Assessment Checklist — <node_id>

### Components (always shown — R7)
| id | name | roots | overview doc | wip |
|----|------|-------|--------------|-----|

### Subgraph
| hop | id | type | wip_status | path from origin |
|-----|----|------|------------|------------------|

### Files loaded
| node | kind | path | status (read / MISSING) |
|------|------|------|--------------------------|

### Constraints inherited from decisions
- <one bullet per decision in hops 0–2: source_ref, what it fixes verbatim, folded ids if any>

### Decisions to re-examine
For each decision D in hops 0–1: affects(D) ∪ resolves(D) ∪ decisions that supersede / are superseded by D
∪ decisions attached (part_of) to the same feature/function.
| decision | why it may drift | related |
|----------|------------------|---------|

### Existing capabilities (R8)
Nodes across ALL components whose name / docs / code_targets overlap the task at hand.
| id | component | what it already provides |
|----|-----------|--------------------------|

### Stale docs (F10)
| node | code last changed | docs last changed | finding |
|------|-------------------|-------------------|---------|

### Blast radius
- Shared code_targets with other nodes: ...
- IN_PROGRESS / BLOCKED nodes in the subgraph: ...

### Pre-modification checks
- [ ] All hop-1 and hop-2 files read (or each MISSING row acknowledged)
- [ ] Existing capabilities reviewed; no duplicate implementation planned
- [ ] Decisions to re-examine acknowledged
- [ ] Stale docs acknowledged (will be updated in this change or logged as unresolved)
- [ ] No conflicting IN_PROGRESS work on shared code_targets
- [ ] current_node set
```

Only after every box can be ticked may code modification begin.

---

## `/graph-handover`

Purpose: persist the exact state of work so a fresh session resumes with zero re-discovery.

1. **Update the graph**: `current_node` (or null if complete); `wip_status` of touched nodes; new edges,
   nodes, `docs`, `code_targets` discovered this session; validate (R1).
2. **Growth check (F2')**: for each feature/function in the subgraph, propose a split when
   (a) attached decision+issue nodes ≥ `config.growth_threshold`, or (b) a decision's scope covers only part
   of the node's `code_targets`, or (c) its design document gained ≥ 2 top-level sections describing separate
   behaviours. On approval: create `function` children with `part_of` the node, move the relevant `docs`,
   `code_targets` and decision/issue attachments to them, leave the parent with overview docs only.
3. **Fold check (F7 = C)**: candidates are (i) a decision that is a `supersedes` target and has no other live
   in-edges, (ii) an issue that is a `resolves` target and has no other live in-edges. Never fold a node with
   `wip_status` IN_PROGRESS or BLOCKED. On approval: survivor.`folded` += folded `source_ref`s;
   survivor.`affects` ∪= folded.`affects`; survivor.`docs` ∪= folded.`docs`; delete the folded node and edges
   to it. Text stays in the registry, so nothing is lost overall.
4. **Staleness (F10)** as in hydrate step 5, over the touched nodes.
5. **Write the handover** to `config.handover_path` (create the directory if needed; overwrite; the graph is
   the durable history, the handover is the live pointer) using the template below.
6. **Fidelity (R3)**: preserve verbatim every explicit technical decision and its reason, every identifier
   chosen or renamed (variable, function, class, file, config key, schema field, enum value, CLI flag), and
   every edge discussed but not resolved. Name the option chosen and the options rejected. Never write
   "refactored X" or "various fixes". Empty sections are written as `- none`.

Growth and fold are proposals in the interaction language; they are never applied silently.

### Handover template

```markdown
# WIP HANDOVER — <project name>
Generated: <YYYY-MM-DD HH:MM> · Graph: `dependency_graph.json` · Schema: `.claude/skills/graph-context-sync/schema/graph_schema.json`

## 1. Active Task Pointer
- current_node: `<node_id>` (`<type>`, wip_status: `<status>`), part_of: `<parent>` → `<component>`
- Name: <node.name>
- Goal in one sentence, as stated by the user: "<verbatim>"
- Work completed this session (file-level):
  - `<path>`: <what changed>
- Work NOT yet done:
  - <item>

## 2. Components and Subgraph Context Range
### Components (R7)
| id | name | roots | overview doc | wip |
|----|------|-------|--------------|-----|
### Subgraph (re-hydrate with `/graph-hydrate <current_node>`)
| hop | id | type | wip_status | why it matters |
|-----|----|------|------------|----------------|
Files read outside the subgraph (candidates to add to the graph):
- `<path>` — <reason>

## 3. Hard Decisions Log
| # | Decision | Chosen | Rejected alternatives | Reason (as stated) | Fixed identifiers | Registry id (if logged) |
|---|----------|--------|-----------------------|--------------------|-------------------|-------------------------|

## 4. Unresolved Edges
| # | From node | To node / file | Kind | What is unresolved | Who or what resolves it |
|---|-----------|----------------|------|--------------------|-------------------------|

## 5. Immediate Resume Trigger
1. Run `/graph-hydrate <current_node>` and confirm the checklist matches section 2.
2. Open `<path>` at `<symbol or line>`; the next edit is: <precise description>.
3. Run: `<command>`; expect: <expected output>.
4. Blocking question for the user, if any: "<verbatim question>"

## 6. Decision Drift
Decisions changed this session and their re-examine sets (see hydrate "Decisions to re-examine").
| changed decision | related node | resolved / unresolved | note |
|------------------|--------------|-----------------------|------|
Growth proposals made this session (accepted / declined): ...
Fold proposals made this session (accepted / declined): ...

## 7. Staleness
| node | code last changed | docs last changed | finding | action |
|------|-------------------|-------------------|---------|--------|
```

## `/graph-compact`

Runs the fold check of `/graph-handover` step 3 on the whole graph on demand, with the same approval
flow. Useful after a batch of registry updates.

---

## Enforced rules

| Rule | Content |
|------|---------|
| **R1 Cross-reference validation** | `nodes[k].id == k` (fix the key, never the id). Every target of `part_of` / `depends_on` / `affects` / `resolves` / `supersedes` and `current_node` exists. No self-edges. `part_of` ≤ 1 and acyclic. `resolves` only decision → issue; `supersedes` only decision → decision. `source_ref` matches some `config.registries[].id_pattern` when registries are defined. Non-component `code_targets` fall under the union of component `code_targets`. Schema-valid (run `python3 -c "import json,jsonschema;jsonschema.Draft202012Validator(json.load(open('.claude/skills/graph-context-sync/schema/graph_schema.json'))).validate(json.load(open('dependency_graph.json')));print('OK')"` when available; otherwise check manually and say so). |
| **R2 Hydration** | If `dependency_graph.json` exists, never modify code before `/graph-hydrate <node_id>` of the relevant node with every pre-modification check ticked. If the node does not exist, create it first (init refresh or manual addition passing R1). If the user explicitly asks to skip, state the risk in one sentence, log the skip in Unresolved Edges, proceed. |
| **R3 Handover fidelity** | Sections 1–7 mandatory; verbatim decisions, identifiers, unresolved edges; empty = `- none`. |
| **R4 Interaction language** | Every question, recommendation table, approval request, proposal (split / fold / component) and checklist shown to the user is written in `config.interaction_language` (inferred from CLAUDE.md and the user's messages when unset). Graph contents, handover file, SKILL text stay English. |
| **R5 Structure follows design docs** | `/graph-init` never generates `task`. Decision / issue nodes exist only when referenced from a document in scope or from registry text of a referenced id. Every decision / issue is attached (`part_of`) to ≥ 1 component / feature / function. |
| **R6 Footprint** | The skill writes only to: its own directory, the six delegating skill directories (`.claude/skills/graph-*/`), `dependency_graph.json`, the file at `config.handover_path`, the marked block in `CLAUDE.md`, the marked block in `.gitignore`. Never design docs, registries, source code, other handover files, `.claude/settings*.json`, or Claude memory. A write outside the footprint is refused and reported. |
| **R7 Always-visible top layer** | Hydrate output and handover §2 begin with the table of all `component` nodes, regardless of hop distance. |
| **R8 No reinvention** | Before proposing any new feature or function, search `nodes` (name, docs, code_targets) across all components and present matches. Never propose a new component autonomously; that is a user decision. Violations are logged in Unresolved Edges. |
