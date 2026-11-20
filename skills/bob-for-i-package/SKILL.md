---
name: bob-for-i-package
description: Load when a task involves IBM Bob Premium Package for i (IBM i modes, skills, tools, Understand, RPG workflows) and you must decide whether Bob Shell can use it, or how to get IBM i work done through Bob Shell.
---

# Bob Premium Package for i, seen from Bob Shell

What this gives the steering agent: a plain answer on IDE-only vs Shell, a catalogue of what the package contains (so you can name it in a brief), and the Shell route that actually works for IBM i questions.

## When to load
- A brief mentions "IBM i Developer mode", "Understand", `/review_rpg`, `read_member`, `execute_sql_statement` or Code for IBM i and you must decide if Bob Shell can do it.
- You plan to send IBM i work (RPG, CL, DDS, Db2 for i) to a `bob run` / `bob chat` worker.
- A Bob worker claims it used a package tool, skill or mode and you need to judge that claim.

## Verdict: IDE-only, not usable from Bob Shell (as documented)
- Every package page lives under `/docs/ide/premium-packages/`. No page says the package works in Bob Shell. No Shell page mentions it (docs).
- It is an IDE extension: requires Bob IDE 2.0.0+ and the VS Code-style extension `IBM/vscode-ibmi-bob` plus Code for IBM i (SSH), Db2 for IBM i, RPGLE, CLLE, IBM i Testing extensions (docs). Tools, modes, skills, `/init` and workflows are registered by that extension inside the IDE host.
- Local check (verified 2.0.6): `bob --help` has only `chat`, `run`, `mcp`, `acp`; no extension, package or premium command. `~/.bob/skills` held no IBM i skills (no `rpg-*`, `db2-*`, `understand-*`). `bob mcp list` shows no package server.
- Treat as: the package's modes, tools, skills, slash commands, workflows, Understand and RAG are NOT available to a Shell worker. The docs give no Shell or ACP route for them; do not promise it in a brief. If you need proof for a specific setup, run a one-off `bob run --format stream-json` asking Bob to list its tools and check for `read_member` or `understand_*` (costs Bobcoins; expect them absent).
- Policy angle: group policies are "supported by Bob IDE only" (docs `ide/security/group-policies`); do not assume they restrict a Shell worker.

## What the package contains (IDE, docs) - use as vocabulary only
| Part | Content |
|---|---|
| Modes | `IBM i Developer` (read, edit, execute, mcp, todo, subtask, skill) and `IBM i Database` (no mcp group). Both inject connection, library list, CCSID and `AGENTS.md` context per task |
| Tools | Read: `read_member`, `search_qsys`, `read_stream_file`, `search_ifs`. Write: `write_member`, `write_stream_file`, `generate_dds`. Execute: `execute_cl_command`, `execute_sql_statement`, `execute_pase_command`, `manage_library_list`, `convert_rpg_source`. Build/test: `get_compile_actions`, `execute_compile_action`, `generate_rpg_unit_test_stub`, `run_rpg_unit_test_suite`. SQL: `get_schema_info`, `get_file_info`, `get_table_columns`, `get_cl_command_library`. Docs: `search_ibm_i_docs_with_rag`, `fetch_cl_command_doc`, `search_sql_examples`, `fetch_sql_example`. Understand: `understand_list_apps`, `_app_summary`, `_find_objects`, `_describe_object`, `_trace_dependencies`, `_find_symbol` |
| Skills (auto-loaded) | About 70: RPG, CL, DDS, COBOL for i, Db2 for i and SQL, IWS, `rpgunit-testing`, plus `understand-app-overview`, `understand-impact-analysis`, `understand-program-explainer` |
| Slash commands | `/init` (writes `AGENTS.md` into `QBOBSRC/AGENTS.MD` or `.bob/AGENTS.md` in the IFS), `/erd`, `/review_SQL`, `/review_rpg` |
| Workflows | RPG Modernization, Business Rules Extraction, RPGUnit Test Plan, RPGUnit Test Suite, SQL Index Strategy Advisor (started from a chat button in the IDE) |
| RAG | `search_ibm_i_docs_with_rag` over IBM i docs and Redbooks |
| Workspaces | Local, Library List, IFS Working Directory |
| Understand | Object scan (not source) of chosen libraries, stored in an install library on the system; dashboard is IDE UI |

Guardrails worth knowing when reviewing IDE output (docs `settings`): `execute_cl_command` needs library-qualified commands; `execute_sql_statement` forces approval for `DROP`, `UPDATE`, `TRUNCATE`, `CALL`, `GRANT` and others; default fetch is 100 rows. The package acts with the connected user profile's full authority, and sets `CLIENT_APPLNAME` for auditing.

## What to do instead from a steering agent
1. Read-only IBM i facts and structure: use Bob Shell with the IBM i tooling you have (ibmi-axi, ibmi-mcp). Load `bob-ibmi-tools`, which detects it first; with Ozgar MCP for an application knowledge graph, add `bob-ibmi-ozgar-mcp`.
2. IBM i knowledge (RPG, CL, Db2 for i syntax): a generic Bob mode plus a clear brief works; the package skills are not there to help. Put the needed conventions into the brief or a `.bob/rules` / `AGENTS.md` file (`bob-brief-and-context`, `bob-config-rules-skills`).
3. Source edits on a local checkout of RPG/CL/DDS: Bob can edit files like any text, but it cannot compile or deploy on the system without a tool for it. Say so in the brief; compile and test elsewhere and verify yourself (`bob-verify-output`).
4. If the user truly needs IDE-only features (Understand scan, `/review_rpg`, RPG Modernization workflow), tell them it needs the Bob IDE and the entitlement; do not fake it in Shell.

## Pitfalls
- Assuming "Bob has IBM i modes" means a Shell worker has them: `--mode` takes only built-in or custom mode slugs (verified 2.0.6 help), and no package mode ships with the Shell; a custom mode named like it would be a persona only, without the tools (inferred).
- Confusing the package tools (`read_member`, SSH via Code for IBM i, IDE) with ibmi-mcp (an MCP server): different mechanisms, different tool names, different auth.
- Accepting a worker's claim "I scanned with Understand" or "I ran RPGUnit": without `tool_use` evidence it is a hallucination in Shell.
- Briefing a write against a live system: the package executes with the full authority of the profile; any substitute path needs a least-privilege, read-only profile.
- Docs pages for Z carry copy text (the package troubleshooting page mentions a "Bob for Z workflow"); do not read it as Z content.

## See also
`bob-ibmi-tools`, `bob-ibmi-ozgar-mcp`, `bob-mcp-setup`, `bob-modes-and-orchestration`, `bob-approvals-and-trust`, `bob-security-and-data`, `bob-devsecops-package`, `bob-verify-output`.

## Sources
`ide/premium-packages/pkg-index`, `.../bob-for-i/bob-for-i-index`, `.../modes`, `.../prerequisites`, `.../tools`, `.../skills`, `.../slash-commands`, `.../workflows`, `.../workspaces`, `.../understand`, `.../retrieval-augmented-generation`, `.../settings`, `.../security-guidelines`, `.../troubleshooting`, `ide/premium-packages/updating-premium-packages`, `ide/security/group-policies`, `shell/features/modes`; `local-help` (bob 2.0.6: `bob --help`, `bob mcp list`, `~/.bob/skills` listing).
