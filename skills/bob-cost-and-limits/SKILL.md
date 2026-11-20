---
name: bob-cost-and-limits
description: Load before running Bob Shell unattended or in parallel, to set --max-cost and --max-turns, read the cost from run output, understand Bobcoins and plan budgets, keep context cost down, and handle BOB_API_KEY safely.
---

# Cost and limits

Cap and read Bob's spend, keep context cheap, and handle the API key without ever exposing its value.

## When to load
- You start `bob run` or `bob chat` without a human watching.
- You plan several Bob tasks, or Bob next to other workers on the same account.
- A run was expensive, stopped early, or failed on authentication.
- You are about to set, pass or check the Bob API key.

## Bobcoins (docs)
- Billing unit; input and output tokens both count, converted to Bobcoins. Monthly allocation resets each billing cycle.
- Included per month: Free trial 50, Pro 50, Pro+ 180, Ultra 1000. Trial lasts 30 days (pricing page). Enterprise buys packs of 1,000.
- Overage is opt-in, billed per Bobcoin beyond the allocation, and once enabled it cannot be turned off until month end. It is an admin or account-owner choice; never enable it on your own.
- The budget is account-wide: every Bob worker, an ibm-bob MCP bridge and the IDE draw from the same pool.
- Observed (evidence, 2.0.6): one minimal `bob run` turn cost about 0.02 Bobcoins; `stats.session_costs` matched the account usage delta. I have no measured numbers for a real multi-file task. Calibrate with your first runs.

## Caps on every unattended run
```bash
bob run --max-cost <BOBCOINS> --max-turns <N> --format json --workspace /abs/workspace "<brief>"
```
- `--max-cost <number>` stops the session when spend reaches it. The docs name the unit Bobcoins; local help gives no unit (verified 2.0.6). Observed `session_costs` is in Bobcoins; Bob's bundled README calls `--max-cost` USD, which does not match that figure.
- `--max-turns <number>` caps agentic turns. Both flags exist on `bob chat` too (verified 2.0.6); what a limit does to a live chat pane was not tested.
- `0` removes the limit; 2.0.6 accepts it (changelog). Never pass `0` unattended.
- Starting values (opinion, not measured): small fix `--max-cost 0.5 --max-turns 15`; feature with tests `--max-cost 2 --max-turns 40`. Raise only after reading real `session_costs`.
- When a limit hits, `stream-json` emits an `error` event "cost/turn limit reached" (docs); `--format json` gives `status: "error"`. The work may be half done: check `git status`, do not assume success.

## Read the cost back (docs)
`--format json` emits one object at the end:
```text
{"type":"result","status":"success|error","stats":{"task_id":"...","total_tokens":0,"input_tokens":0,"output_tokens":0,"cache_read_tokens":0,"cache_ratio":0,"duration_ms":0,"session_costs":0,"tool_calls":0},"last_message":"..."}
```
- Log `stats.task_id` (needed for `--resume`, see `bob-sessions-resume`), `session_costs` and `tool_calls` per task. Sum them for a campaign.
- A high `tool_calls` with low progress means looping or poisoned context: stop, fresh session.
- `stream-json` for live events (`message`, `tool_use`, `tool_result`, `error`, `result`).

## Balance and plan checks
- No CLI usage command: `bob --help` lists only `chat`, `run`, `mcp`, `acp` (verified 2.0.6). Interactive `/status` shows monthly budget and usage and session balance (docs). The Bob administration portal shows usage and plan.
- An undocumented read-only account endpoint exists. Do not build your own client on it.
- Before a campaign: compute (tasks x cap) and compare to the remaining budget. Opinion: keep half the remaining budget unplanned.

## Keep each prompt cheap
- Bob re-sends the whole active context on every prompt; long sessions get more expensive per turn (docs).
- Baseline is about 8,500 tokens for a trivial prompt; compaction starts near 190k of the 270k window and is lossy; 20k is reserved for the reply.
- Levers (docs unless marked): one goal per session; named files, not directories; short `AGENTS.md`; `--disable-mcp` or disable unused MCP servers and skills; `explore` subagents run on a lighter model; revert and restart instead of correcting a bad run in place; paste only needed log lines.
- Opinion: use `--disable-subagents` for tiny tasks, since spawn overhead exceeds the gain.
- Prefer plan or ask mode for exploration; they cannot trigger long edit and test loops.

## API key handling (names only)
- Headless auth: environment variable `BOB_API_KEY`. the docs say `bob run` fails with an authentication error without it and that SSO tokens from `bob chat` are not reused; on 2.0.6 a run with no key variable succeeded on a configured host that had a credential stored in Bob settings (verified, the TUI said "Using API key authentication"), so treat the variable as required for CI and fresh hosts, not as the only path. An older name `BOBSHELL_API_KEY` appears in ibm-bob MCP bridge docs and tooling; use one name per process, and never set both to different values.
- Key types: `general` (needs `--team-id <TEAM_ID>` when running; verified flag in 2.0.6 help) and `inference` (scoped to one team, no extra header). Created and revoked only at bob.ibm.com, shown once (docs).
- Check presence without printing it:
```bash
[ -n "${BOB_API_KEY:-}" ] && echo key-set || echo key-missing
```
- Pass it in the process environment, ideally from a launcher that sources a mode-600 file. Never as a command-line argument (visible in `ps`), never in a brief, a commit, a log, a JSON report or a `bob run --format json` capture you store.
- Every tool process Bob starts inherits that environment, so Bob can read the key. Use a dedicated key, do not export it globally, and do not run Bob on untrusted input with a high-value key (opinion).
- If the key may have leaked, report it to the owner; rotating it is the owner's decision.

## Pitfalls
- Unattended run with no cap: one confused loop can burn a day's budget. Always set both caps.
- Treating `--max-cost` as dollars: it is Bobcoins; mixing the two gives caps 2x off or worse.
- Parallel workers each with their own cap still share one account budget; total can exceed what you meant.
- Session history grows cost per turn; a 40-turn chat costs more than four 10-turn tasks.
- Auth failure looks like a run failure; read `status` and the first error line before retrying (retries cost startup tokens).

## See also
`bob-run-headless`, `bob-sessions-resume`, `bob-brief-and-context`, `bob-modes-and-orchestration`, `bob-tools-overview`, `bob-security-and-data`, `bob-troubleshooting-pitfalls`.

## Sources
`shell/account/bobcoins`, `shell/account/api-keys`, `shell/getting-started/start-bobshell-non-interactive`, `shell/getting-started/install-and-setup`, `shell/core-concepts/context-window-management`, `shell/getting-started/best-practices`, `shell/features/subagents`, `shell/changelog`; `local-help` (`bob --help`, `bob run --help`, `bob chat --help`, 2.0.6).
