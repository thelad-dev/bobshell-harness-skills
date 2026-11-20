---
name: bob-when-to-use
description: Load before delegating a task to IBM Bob Shell, or when choosing between Bob and another worker. Decision guide on what Bob is good for, what to keep away from it, and the preflight checks before dispatch.
---

# When to use Bob Shell

Decide whether Bob is the right worker for a task, and run the preflight checks before you dispatch it.

Labels: **evidence** = from docs, local `bob` 2.0.6 or a recorded run; **opinion** = judgment, no measurement behind it.

## When to load
- You are about to hand a task to Bob (`bob run`, `bob chat`, ACP, an ibm-bob MCP bridge).
- You must pick between Bob and another worker (Claude, Codex, Grok, Cursor).
- A Bob run failed or looks wasteful and you question the choice.

## What Bob has (evidence)
- Headless one-shot (`bob run`) and interactive (`bob chat`) entry points, plus `bob acp` and `bob mcp` (verified 2.0.6).
- Modes agent / plan / ask, custom modes, subagents, skills, hooks, MCP client, `AGENTS.md` and `.bob/rules-*` loading.
- Worker runs observed on 2.0.6, 2.0.9 and 2.1.0: read a brief from a file, created a file, committed, reported done. The tasks were small. There is no evidence for large refactors or long multi-hour runs.
- It bills on its own account budget (Bobcoins), separate from your other providers. See `bob-cost-and-limits`.
- No `--model` or `--effort` flag (verified 2.0.6): you cannot choose model or reasoning depth per task.

## Good fits
- IBM i questions and code (RPG, CL, DDS, COBOL, Db2 for i) when IBM i tooling is available to Bob (ibmi-axi, ibmi-mcp or Ozgar MCP). Opinion: Bob is a sensible host because IBM ships IBM i personas and packages for it; the package docs are IDE pages, so confirm what the Shell actually sees (the in-session MCP tool list, `command -v ibmi-axi`, `/mode`; `bob mcp list` alone is not proof). See `bob-ibmi-tools`, `bob-ibmi-ozgar-mcp`, `bob-for-i-package`.
- Read-only work: code survey, review, explanation, plan files. Use ask or plan mode. See `bob-modes-and-orchestration`.
- Self-contained implementation with a clear done-condition (a test command, a file to produce). Use agent mode.
- Spare capacity: another budget to burn when your own quota is tight (opinion, check Bob's balance first).
- Parallel independent tasks in separate workspaces, if the shared monthly budget allows.

## Poor fits
- Trivial edits (rename, one-line fix, config tweak). A single turn costs about 0.02 Bobcoins and an empty prompt already loads about 8,500 tokens (evidence). Overhead of brief and verification exceeds the work. Opinion: do it yourself.
- No key, no budget, or expired plan. The docs say `bob run` needs `BOB_API_KEY` and does not reuse an SSO login; on 2.0.6 a run without the variable did work on a host with a credential stored in Bob settings (verified). A host with neither fails after startup cost, so check before dispatching.
- Work that needs files outside the workspace. Bob blocks reads outside its workspace by default. Copy inputs in first.
- Tasks that need you to pick model, effort or temperature. No such flag exists.
- Destructive or ambiguous work in a headless run: `bob run` pre-approves all tools, so there is no approval gate (docs). Keep such work out of it, or restrict tools (`bob-tools-overview`).
- Secrets-heavy or regulated repos before you read `bob-security-and-data` (opinion: check data residency and telemetry first).
- Steering that needs tight, token-level control of the loop. Bob runs its own agent loop; you only brief, interrupt and read results.

## Preflight before every dispatch
```bash
bob --version                              # docs ask for 2.0.9+; 2.0.6 still works locally
[ -n "${BOB_API_KEY:-}" ] && echo key-set  # prints no value; never echo the key
bob mcp list                               # which MCP servers this workspace would load
```
- The workspace is trusted, or you pass `--trust` deliberately (`bob-approvals-and-trust`).
- The task has an input list (`@paths`), a done-condition, and a mode (`bob-brief-and-context`).
- A cap is set: `--max-cost` and `--max-turns`.
- The `bob` on PATH is a 2.x install. A stale 1.0.x `bob` rejects `chat --auto-approve --workspace` with "Unknown arguments" (verified).

## Quick pick
| Task | Choice |
|---|---|
| Explain or survey code, no edits | Bob ask mode, or do it yourself if it is 1-2 files |
| Design before coding | Bob plan mode, then a fresh agent task |
| Implement with tests | Bob agent mode, capped, verified by you |
| Question on an IBM i program or file | ibmi-axi, ibmi-mcp or Ozgar MCP directly or via Bob (`bob-ibmi-tools`); without any of them say so, do not guess |
| One-line fix | Do it yourself |

## Pitfalls
- Expecting a model choice: there is no flag and the docs FAQ answer is not in the fetched text. Opinion: assume the model can change server-side. Pin behavior with tests, not with the prompt.
- Treating Bob's final message as proof: it is a claim. Verify with `bob-verify-output`.
- Parallel Bob workers all draw from one account budget; five parallel runs can drain it fast.
- Bob auto-updates itself via `npm install -g --force` on start, which can change the version mid-campaign. Check `bob --version` per run.

## See also
`bob-shell-steering`, `bob-brief-and-context`, `bob-modes-and-orchestration`, `bob-cost-and-limits`, `bob-run-headless`, `bob-verify-output`, `bob-troubleshooting-pitfalls`.

## Sources
`shell` (index), `shell/getting-started/best-practices`, `shell/getting-started/start-bobshell-non-interactive`, `shell/core-concepts/context-window-management`, `shell/changelog`; `local-help` (`bob --help`, `bob run --help`, `bob chat --help`, `bob mcp list`, 2.0.6).
