---
name: bob-shell-steering
description: Load first whenever you, as an agent, are about to start, brief, supervise or verify IBM Bob Shell (bob chat, bob run, bob acp) as a tool or worker. Routes to the other bob-* skills and gives the ten rules that matter most.
---

# Steering IBM Bob Shell (entry point)

Entry skill for an agent that drives Bob Shell (CLI `bob`, npm `@ibm/bob`) as a worker. It tells you which `bob-*` skill to load next and the rules that prevent the common failures. Facts were checked against the IBM docs (fetched 2026-11-18) and a local Bob 2.0.6; unmarked = docs, `(verified 2.0.6)` = observed.

## When to load
- You plan to hand any task to Bob, or you already run a Bob pane or process.
- You are unsure which Bob entry point (`chat`, `run`, `acp`) or which mode fits.
- A Bob run produced something you must judge or repair.

## Route: situation to skill
| Situation | Load |
|---|---|
| Is Bob the right worker? Preflight before dispatch | `bob-when-to-use` |
| Start a long-lived Bob in a terminal pane, deliver the brief, interrupt, exit | `bob-launch-interactive` |
| One bounded task, JSON output, scripts and CI | `bob-run-headless` |
| Structured multi-turn control without screen scraping | `bob-acp-integration` |
| Continue, list or export a task; avoid pickers | `bob-sessions-resume` |
| Write the brief, cut the task, choose context | `bob-brief-and-context` |
| Pick mode, restrict tools, subagents, staging | `bob-modes-and-orchestration`, `bob-tools-overview` |
| Spend, caps, API key handling | `bob-cost-and-limits` |
| Autonomy level, trusted folders, "auto-approve does nothing" | `bob-approvals-and-trust` |
| Secrets, ignore files, data residency, telemetry | `bob-security-and-data` |
| Busy/idle tracking, guards, delivery proof | `bob-hooks` |
| Settings, rules, `AGENTS.md`, installing skills for Bob | `bob-config-rules-skills` |
| Give Bob MCP servers | `bob-mcp-setup` |
| IBM i questions (optional: ibmi-axi, ibmi-mcp, Ozgar MCP) | `bob-ibmi-tools`, plus `bob-ibmi-ozgar-mcp` for an Ozgar knowledge graph |
| Does the IBM i or DevSecOps package exist in the Shell? | `bob-for-i-package`, `bob-devsecops-package` |
| Judge a finished run, prove tool use, check the diff | `bob-verify-output` |
| Something stalls, fails or behaves oddly | `bob-troubleshooting-pitfalls` |

## Ten rules
1. Pick the entry point by control need: `bob run -f json` for one bounded task; `bob chat` in a pane for a worker you watch and steer; `bob acp` when a program must drive it.
2. Always pass `--workspace <absolute path>`, absolute `bob`, and `--trust` or a pre-registered trusted folder. In an untrusted folder `--auto-approve`, hooks, project skills and MCP are silently dropped. Even in a trusted folder two dialogs still ask; watch the pane (`bob-approvals-and-trust`).
3. There is no `--model` or `--effort` flag (verified 2.0.6). Do not promise a model; pin outcomes with tests.
4. `bob chat` has no positional prompt (verified 2.0.6). Put the brief in a file inside the workspace and send one line: `read <file> and follow it exactly.`
5. In `bob run` all tools are pre-approved (docs) and a run without the key variable can still work on a host where Bob has a stored credential (verified 2.0.6). Constrain with `--mode ask|plan`, `--disable-tool-groups`, a clean git checkout and caps.
6. Set `--max-cost` (Bobcoins, not dollars) and `--max-turns` on every unattended run; the budget is shared by all Bob workers of an account.
7. Close stdin for `bob run` with an argument prompt (`</dev/null`); an open stdin hung a run for minutes (verified 2.0.6).
8. Brief = goal, exact inputs (`@path`), constraints, done-condition, report format. One goal per task; start a fresh task rather than correcting a drifting one.
9. Bob's final message is a claim. Verify with the diff, the tests and tool-call evidence (`bob-verify-output`).
10. Never put a key, token or bearer header in a brief, argv, log or skill. Do not print `~/.bob/settings.json` (it can hold one).

## Minimal working lines
```bash
# One-shot, read-only, JSON result (verified 2.0.6)
bob run -f json --mode ask --trust --max-turns 6 --max-cost 0.5 --workspace /abs/ws "Question. Cite file:line." </dev/null

# Worker in a pane (verified 2.0.6)
env -u BOB_SESSION /abs/bob chat --auto-approve --trust --workspace /abs/ws --mode agent
```
Then read `stats.task_id`, `stats.session_costs` and `last_message` from the JSON, or watch the pane.

## Selecting skills for a dispatch (for a rules file)
- Always: `bob-shell-steering`.
- Spawning a Bob pane: add `bob-launch-interactive`, `bob-approvals-and-trust`, `bob-hooks`, `bob-brief-and-context`.
- One-shot or CI: add `bob-run-headless`, `bob-cost-and-limits`.
- Read-only survey or review: add `bob-modes-and-orchestration`, `bob-verify-output`.
- IBM i work (only if IBM i tooling is installed): add `bob-ibmi-tools`, `bob-mcp-setup`; with Ozgar MCP also `bob-ibmi-ozgar-mcp`.
- On trouble: `bob-troubleshooting-pitfalls`.
Skills meant for the worker itself (copy into `<workspace>/.bob/skills/`): only `bob-ibmi-ozgar-mcp`, and only with Ozgar MCP. The others describe how to steer Bob and are of no use inside it.

## See also
All other `bob-*` skills in this set.

## Sources
`shell` (index) and all `shell/*` pages listed in the set's sources table; `local-help` (`bob --help`, `bob chat|run|mcp|acp --help`, 2.0.6); live runs on 2.0.6 (skill discovery, stream-json shape, project MCP).
