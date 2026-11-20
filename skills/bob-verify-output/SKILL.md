---
name: bob-verify-output
description: Use after Bob Shell reports a task done, fails, or goes quiet, to check the real result (git diff, tests, json/stream-json events, cost and turn caps, stuck dialogs, partial runs) instead of trusting Bob's own summary.
---

# Verifying Bob Shell output

Gives the steering agent a fixed check order after Bob says "done": trust the repo and the test run, use Bob's output only as supporting evidence.

## When to load
- Bob (`bob run` or an interactive `bob chat` worker) just reported completion and you are about to merge, report, or build on it.
- A run ended early, hung, hit a limit, or the pane looks idle but nothing happened.
- You parse `bob run -f json` or `-f stream-json` output in a script.

## Rule: the diff is the truth
Bob's `last_message` is a claim. Check in this order, stop at the first failure:

```bash
git -C <ws> status --short                      # uncommitted work
git -C <ws> log --oneline <base>..HEAD          # Bob may have committed itself
git -C <ws> diff --stat <base>                  # what changed, incl. commits
git -C <ws> diff <base> -- <paths-in-brief>     # read the actual change
```
- Bob commits on its own when the brief allows it. A clean `git status` after "done" does not mean no work: look at `git log <base>..HEAD` (verified 2.0.6, worker run: file created, committed, `done`, `git status` empty).
- Check scope: files changed outside the brief, new files under `.bob/`, edited settings or rules files. Revert or flag them.
- Re-run the tests, linter and build yourself, in your shell, with your own command. Never accept "tests pass" from Bob's text.
- For read-only briefs (`--mode ask`, `--mode plan`, `--disable-tool-groups edit,execute`), assert the tree is unchanged: `git status --short` must be empty. Plan mode may still edit Markdown and text files (docs, `shell/core-concepts/tools`).

## Reading `bob run -f json`
Docs: one object printed only after the session completes. Fields: `type` ("result"), `timestamp`, `status` ("success" or "error"), `last_message`, and `stats` with `task_id`, `total_tokens`, `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_tokens`, `cache_ratio`, `duration_ms`, `session_costs`, `tool_calls`.

```bash
bob run -f json --max-turns 15 --max-cost 2 --workspace <ws> "<brief>" </dev/null > result.json
jq '{status, tool_calls: .stats.tool_calls, cost: .stats.session_costs, ms: .stats.duration_ms, task: .stats.task_id, msg: .last_message}' result.json
```
- `status == "error"` or an empty file: the run failed, do not parse `last_message` as a result.
- `stats.tool_calls == 0` on a task that must edit files: Bob only talked. Treat as not done.
- `stats.session_costs` is in Bobcoins for this task only (verified 2.0.6: matches the account usage delta). Compare it to the cap you passed; a value at or near `--max-cost` means the cap probably stopped the run.
- Keep `stats.task_id`: it is the handle for `bob run --resume <task-id>` (see `bob-sessions-resume`).
- Close stdin (`</dev/null`) or the run can hang with no output (verified 2.0.6, 150 s idle). See `bob-run-headless`.
- A killed or timed-out run emits no json at all (the object comes at the end). For long tasks use `-f stream-json` so partial evidence survives.

## Reading `-f stream-json` (NDJSON)
Docs list five event types, one JSON object per line, discriminated by `type`:

| type | key fields |
|---|---|
| `message` | `role`, `content`, `isReasoning?` |
| `tool_use` | `tool_name`, `tool_id`, `parameters` |
| `tool_result` | `tool_id`, `status`, `output?`, `error?` |
| `error` | `severity`, `message` (docs: cost or turn limit reached) |
| `result` | `status`, `stats`, `last_message` (final summary) |

Verified on 2.0.6 with a live run: the `message` event with `role: "assistant"` arrives in many small fragments (a few characters each, mid-word), so join the `content` values in order to read the answer; `role: "user"` echoes your prompt once. `tool_use` carried `tool_name`, `tool_id`, `parameters`; `tool_result` carried `tool_id`, `status`, `output`; the last line was `result` with `stats` (`task_id`, `duration_ms`, `session_costs`, `max_cost`, `tool_calls`) and, in `-f json` only, `last_message`. Every event has a `timestamp`. Capture a sample from a tiny task before you write jq for other event kinds (`error` was not provoked):
```bash
bob run -f stream-json --max-turns 2 --workspace <ws> "List the files in the repo root" </dev/null > sample.ndjson
jq -c '{type, tool_name, status}' sample.ndjson
```
Then derive evidence:
- Tool evidence: every claimed edit needs a `tool_use` (write/edit tool names are `write_file`, `apply_diff`, `insert_content`, `search_and_replace`, `execute_command`; docs `shell/core-concepts/tools`) followed by a `tool_result` that is not an error. Cross-check with the diff.
- No final `result` line means the run was cut off (kill, crash, timeout). Treat as partial.
- An `error` event or `result.status == "error"` means stop and read `message`; do not retry blindly (see `bob-cost-and-limits`).
- In non-interactive runs all tools are pre-approved (docs), so `tool_result` errors are real failures, not approval prompts. Blocked outside-workspace calls are explained in the message (docs 2.0.6).
- Do not grep the `pretty` format for tool names: the UI is translated by locale (live 2.1.0 de_DE showed "Datei ... schreiben (abgeschlossen)" instead of "Write File (completed)").

## Interactive workers (`bob chat`)
There is no result object. Evidence sources, strongest first: the repo diff, hook records, `/export`, then the screen.
- Turn end: a `Stop` hook in `.bob/settings.json` fires when the agent stops (docs); a `UserPromptSubmit` hook fires on submit. Bob has no `StopFailure` or `SessionEnd` hook, so a failed turn (auth error, quota, crash) leaves your busy record at busy, and a single Escape interrupt does not fire `Stop` either (verified 2.0.6). "Busy forever" is not proof of work: look at the pane and the log.
- Hooks only load in a trusted folder; in an untrusted folder `.bob/settings.json` is skipped silently, so you get no turn signal at all (docs, verified 2.0.6). See `bob-approvals-and-trust`.
- Transcript: `/export json <file>` or `/export markdown <file>` writes the active task into the workspace (docs 2.0.6). Task list: `bob --list-tasks` (NDJSON when not a TTY; fields `id`, `title`, `status` active|completed|paused, `updatedAt`).
- Screen reading: the idle composer is a `❯` row between two `─` rules with a grey placeholder. The placeholder wording is translated, so identify "empty" by the grey styling, not by the words (verified 2.0.6 and 2.1.0). Bob hides the terminal cursor.

## Stuck or blocked states (nothing happens, no error)
Capture the visible pane and look for these before assuming Bob is working:
- Licence dialog ("IBM Licence Agreement ... press y to accept"): fresh `HOME` or new version. Needs top-level `licenseConsent: true` in settings or `--accept-license` (verified 2.1.0). Never type into a pane whose composer is not recognised.
- Trust dialog (Trust folder / Trust parent / Don't trust): only if the Folder Trust setting is on. `--trust` marks the folder (docs).
- Mandatory team picker after an invalid `--team`/`--instance`: exit with Ctrl+C; Esc does not dismiss it (docs 2.0.2).
- Auth failure or exhausted budget: the footer or last lines show it; the hook state still says "working" (verified 2.0.6).
- A per-tool approval prompt sitting open: `--auto-approve` is suppressed in an untrusted folder, so Bob waits for a human forever (docs, verified 2.0.6).

## Partial runs
1. Snapshot the repo state first (`git status`, `git diff > partial.patch`).
2. Decide: keep Bob's partial work and `bob run --resume <task-id> "<what is missing>"`, or `git restore`/reset and re-brief smaller.
3. Resume brief names only the remaining delta and the failing check, not the whole task again.

## Pitfalls
- Trusting `last_message` over the diff: Bob can describe changes it did not make or tests it did not run. Always re-run tests yourself.
- Parsing `-f json` from a run you killed: the file is empty or truncated, you may read "no error" as success.
- Reading `git status` only: a clean tree after a self-commit hides the work, and a dirty tree can hold Bob's scratch files.
- Treating `session_costs` as USD: it is Bobcoins (verified 2.0.6).
- Assuming exit code 0 means success: exit-code behaviour for limit hits and errors is unverified; read `status` and the events.
- Depending on `pretty` text or localized TUI strings: they change with `LC_ALL`/`LANG` and the `locale` setting.

## See also
`bob-run-headless`, `bob-sessions-resume`, `bob-cost-and-limits`, `bob-approvals-and-trust`, `bob-hooks`, `bob-troubleshooting-pitfalls`.

## Sources
`shell/getting-started/start-bobshell-non-interactive`, `shell/core-concepts/tools`, `shell/configuration/lifecycle-hooks`, `shell/security/trusted-folders`, `shell/changelog`, `shell/troubleshooting/troubleshoot`; local-help (`bob run --help`, `bob chat --help`, `bob --help`, 2.0.6).
