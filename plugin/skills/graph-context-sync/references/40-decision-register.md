# Decision register (public)

Design decisions behind the protocol, in the order they were made. Folded = absorbed into a later decision
(kept here for traceability). Dates: 2026-09-24 unless noted.

| id | Decision | Supersedes / resolves | Status |
|----|----------|-----------------------|--------|
| D1 | v1: closed node objects, open root | — | folded into D13 |
| D2 | Node ids pattern-constrained (`^[a-z0-9][a-z0-9_-]*$`), applied to ids, keys, `current_node`, edges | — | active |
| D3 | JSON Schema draft 2020-12 | — | active |
| D4 | Ship a minimal seed template | — | active |
| S1 | `dependency_graph.json` lives at the project root | — | active |
| S2 | v1: single SKILL.md, `/graph-*` as sub-commands | — | folded into D16 |
| S3 | init scans design docs + referenced code, no blind source walk | — | folded into D17 |
| S4 | Skill text and stored artifacts in English | — | active |
| S5 | Every edge target must exist | — | folded into D18 (now rule R1) |
| D5 | Registry mapping lives in `config.registries`; project CLAUDE.md carries prose nuance + a pointer | resolves OP1 | active |
| D6 | Bounded footprint: fixed path list, marked blocks, install/uninstall, sha256 snapshot, rule R6 | — | active (footprint shrunk by D20) |
| D7 | Compaction = fold superseded decisions / resolved issues into the survivor via `folded[]`; proposal-only | — | active |
| D8 | Typed edges `resolves` and `supersedes` | — | active |
| D9 | `growth_threshold` default 5, tunable; init idempotent, preserves human decisions; `--reset-structure` | — | active |
| D10 | `source_ref` single-valued on decision/issue nodes only; growth measured on the feature side | resolves OP2 | active |
| D11 | `code_roots` not in config; derived from component nodes | resolves OP3 | active |
| D12 | Component layer under Root; R7 components always visible; R8 no reinvention | — | active |
| D13 | Root closed: `current_node`, `nodes`, `config` only | resolves OP4; supersedes D1 | active |
| D14 | Staleness check (code newer than docs, missing paths, unknown `source_ref`) | — | active |
| D15 | First application target: the skill's own repo, then a real project | — | active |
| D16 | Thin delegating skills per command | supersedes S2 | active (renamed by D21) |
| D17 | Component detection = top-level directory listing; S3 governs `code_targets` derivation | supersedes S3 | active |
| D18 | S5 absorbed into R1 | supersedes S5 | active |
| D19 | Handover template hardening: legend, "Resolved this session" line, who/when, local-only marker | — | active |
| D20 | Package as a Claude Code plugin (marketplace + `plugin/`); skill files never enter the project; paths via `${CLAUDE_SKILL_DIR}` / `${CLAUDE_PLUGIN_ROOT}`; footprint = graph + handover + 2 marked blocks | refines D6 | active |
| D21 | Delegate skills renamed to `install/uninstall/init/hydrate/handover/compact` → `/graph-context-sync:<name>`; bare `/graph-*` is impossible for plugin skills | refines D16 | active |
