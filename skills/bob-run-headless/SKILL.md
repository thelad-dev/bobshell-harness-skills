---
name: bob-run-headless
description: Load when you delegate one bounded task to Bob Shell non-interactively with `bob run`, need JSON or stream-json output, caps on cost and turns, a prompt via argument or stdin, or an API key setup for scripts and CI.
---

# One-shot Bob: `bob run`

Run Bob without a TUI, get structured output, parse the result. Unmarked facts are docs; `(verified 2.0.6)` = local `bob run --help` or a live run on Bob 2.0.6.

## When to load
- One self-contained task with a clear end: review, analysis, small edit, doc generation.
- You want machine-readable output (final message, cost, task id) instead of scraping a pane.
- You run from a script, CI, or another agent's shell tool.
- For multi-turn work, steering by keystrokes or live approvals use `bob-launch-interactive` or `bob-acp-integration` instead.

## Command
```
BOB_API_KEY=... bob run -f json --max-turns 20 --max-cost 5 \
  --workspace /abs/repo --mode agent --trust \
  "Do X. Touch only Y. Report Z." </dev/null > /abs/out/result.json
```
- `bob run [options] [prompt...]`. The prompt is positional (verified 2.0.6). `-p` is NOT a `run` option (verified 2.0.6 help; a README says otherwise, ignore it). The root command `bob -p <prompt>` exists in 2.0.6 help, behaviour untested.
- Close stdin with `</dev/null` when you pass the prompt as an argument. With stdin left open `bob run` produced nothing for 150 s (verified 2.0.6).
- Prompt from stdin instead: `bob run -f json < /abs/prompt.md` or `cat prompt.md | bob run` (docs; best for multi-line prompts, no shell quoting). Pipe input can also be prepended: `cat err.txt | bob run "Explain this error"`.
- Prompt may follow flags without values such as `--disable-mcp` (fixed in 2.0.6).
- `@path` in the prompt inlines a file (100 KB per file, 500 KB total) relative to the workspace.
- Always pass `--workspace` with an absolute path; calls outside it run only if global settings and the permission group allow (2.0.6 changelog).

| Flag (verified 2.0.6) | Use |
|---|---|
| `-f, --format pretty\|json\|stream-json` | Output format. Use `json` for one result, `stream-json` to follow progress. |
| `--max-cost <n>` | Stop at n Bobcoins (not USD). `0` = no cap. |
| `--max-turns <n>` | Stop after n agentic turns. `0` = no cap. |
| `--mode <slug>` | `agent` (default), `plan`, `ask`, custom. `ask` makes no changes (docs). |
| `-w, --workspace <path>` | Workspace root. |
| `--trust` | Mark the folder trusted for this and later runs (persists). |
| `--disable-mcp`, `--disable-subagents` | Cut MCP servers or subagents; faster and cheaper for small jobs. |
| `--disable-tool-groups <list>` | e.g. `execute,mcp` for a no-shell, no-MCP run. |
| `-r, --resume [task-id]` | Continue a task; see `bob-sessions-resume`. |
| `--team-id <id>` | Required with a `general` API key. |
| `--accept-license` | Skip the license prompt (needed on a fresh `HOME`). |
| `--log-level <lvl>` | `debug` writes request logs; useful once, noisy otherwise. |

Docs also list `--command-security` and `--config-dir` for `run`; 2.0.6 help does not have them (use 2.0.9+, check `bob run --help`).

## Auth and environment
- Docs: `bob run` needs an API key in env var `BOB_API_KEY` (older name `BOBSHELL_API_KEY`; Bob refuses to start if both are set to different values) and SSO tokens from `bob chat` are not reused. Observed 2.0.6: a `bob run` with neither variable set succeeded on a configured host (task cost 0.04 Bobcoins) and the TUI there reported "Using API key authentication", so Bob had a credential stored in its own settings. Do not depend on that for CI or fresh hosts; set the key. Missing key = authentication error exit.
- Create the key with scope Inference. A `general` key needs `--team-id <TEAM_ID>` too.
- Keep the key out of argv, logs and the prompt. Load it from a 600 file inside a launcher or subshell; never `echo` it.
- Isolated experiment: `HOME=/abs/scratch bob run ...` with `bobShell.autoUpdate=false` in that HOME's `.bob/settings/settings.json` (verified 2.0.6). Fresh HOME needs `--accept-license`.
- Needs Node 24+. Bob may self-update with `npm install -g --force` at start; turn it off in user settings (verified 2.0.6).

## Approvals: none
All tools are pre-approved in `bob run` (docs). There is no `--auto-approve` flag for `run` (verified 2.0.6). The guard is the mode, `--disable-tool-groups`, trust, and a git-clean checkout you can diff. See `bob-approvals-and-trust`.

## Reading the result
`-f json` prints one object at the end:
`{"type":"result","status":"success|error","stats":{"task_id","total_tokens","duration_ms","session_costs","tool_calls",...},"last_message":"..."}`
```
jq -r '.status, .stats.session_costs, .stats.task_id' result.json
jq -r '.last_message' result.json
```
- Keep `stats.task_id` to resume later. `stats.session_costs` is Bobcoins for this task (verified 2.0.6: 0.020858 for a minimal turn).
- `-f stream-json` prints NDJSON events: `message`, `tool_use`, `tool_result`, `error` (cost/turn limit reached), `result` (final). Take the last line with `"type":"result"`.
- Do not trust `last_message` alone. Check the diff and run tests yourself: `bob-verify-output`.
- Exit codes are not documented and were not tested; branch on `status`, and on a missing result object treat the run as failed.

## Pitfalls
- No `--max-cost` and `--max-turns` on an open task can burn the monthly Bobcoin budget; always set both for unattended runs. Docs call `--max-cost` USD, the unit is Bobcoins.
- A limit stop emits an `error` event (docs) and leaves partial edits on disk (inferred). Inspect the tree before retrying.
- stdin left open hangs forever (see above). In CI also set a wall-clock `timeout`.
- Docs combine piped stdin with a prompt argument (`cat err.txt | bob run "Explain"`); for deterministic prompts send one source.
- `--trust` marks the folder trusted (docs: persistent for `chat`); do not pass it for repos you do not control.
- Use an absolute path to `bob`; a non-login shell may lack your PATH.
- If an `ibm-bob` MCP bridge (`bob_prompt`) is installed it needs `BOBSHELL_API_KEY` and names modes `advanced|plan|code|ask`; those names differ from 2.0.6 modes (`agent`, `plan`, `ask`). Prefer `bob run` for control.

## See also
`bob-launch-interactive`, `bob-sessions-resume`, `bob-cost-and-limits`, `bob-modes-and-orchestration`, `bob-approvals-and-trust`, `bob-verify-output`, `bob-brief-and-context`.

## Sources
`shell/getting-started/start-bobshell-non-interactive`, `shell/getting-started/install-and-setup`, `shell/getting-started/bobshell-examples`, `shell/changelog`, `local-help` (`bob run --help`, bob 2.0.6).
