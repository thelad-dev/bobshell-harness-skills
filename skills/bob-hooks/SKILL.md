---
name: bob-hooks
description: Load when wiring Bob Shell lifecycle hooks, for example to track busy/idle state of a Bob worker, confirm prompt delivery, log tool calls, or block commands with a PreToolUse guard.
---

# Bob lifecycle hooks

Hooks are the only reliable machine-readable state signal an automation harness gets from an interactive Bob. This covers events, config, exit codes and the busy/idle pattern.

## When to load
- Building or debugging busy/idle detection for a Bob pane.
- Needing proof that a prompt was delivered or a turn ended.
- Adding a guard that denies certain tool calls while auto-approve is on.
- A hook "does nothing" (usually trust or scope).

## Events (docs)
| Event | Fires | Blocks | Stdout |
|---|---|---|---|
| `SessionStart` | once, new session (`source`: startup, resume, compact) | no | injected into context |
| `UserPromptSubmit` | each submitted prompt, before the model | exit `2` | injected into context |
| `PreToolUse` | before a matched tool | exit `2` or JSON deny | optional JSON (deny, `updatedInput`) |
| `PostToolUse` | after a tool succeeded (not on error) | no | appended to tool result |
| `PreCompact` / `PostCompact` | around compaction | exit `2` on Pre | ignored |
| `Stop` | after the final turn of the agent | no | ignored |

Stdin is one JSON object: `session_id`, `cwd`, `hook_event_name`, plus `prompt` (UserPromptSubmit), `tool_name`/`tool_input`/`tool_use_id` (Pre/PostToolUse), `last_assistant_message` (Stop), `trigger` (compaction).

## Config shape and scope
```json
{
  "hooks": {
    "UserPromptSubmit": [
      { "hooks": [ { "type": "command", "command": "<abs-path>/bob-hook busy", "timeout": 5 } ] }
    ],
    "Stop": [
      { "hooks": [ { "type": "command", "command": "<abs-path>/bob-hook idle", "timeout": 5 } ] }
    ]
  }
}
```
- Fields: `type` (`command` or `http`, https URLs only), `command` (run as `sh -c`, cwd = task folder), `url`, `matcher` (regex on tool name for Pre/PostToolUse, `source` for SessionStart, `trigger` for compaction; omit or `*` for all), `timeout` seconds (default 10, must be above 0).
- Scopes: global `~/.bob/settings/settings.json` (always runs, also in unresolved folders) and workspace `.bob/settings.json` (merged on top, only in trusted folders, silently skipped when untrusted). Use workspace scope for per-task hooks so nothing leaks to other sessions.
- `/hooks` in a session lists every hook with event, matcher, scope, status and command, and toggles them. Use it to check that the hook loaded.
- Edit the file before Bob starts; if the project already tracks its own `.bob/settings.json`, merge instead of overwriting, and keep your additions out of git (`.git/info/exclude`). A `.bob` symlink would redirect the write outside the worktree.

## Exit codes
`0` ok. `2` blocks, but only for `UserPromptSubmit`, `PreToolUse`, `PreCompact`; stderr (else stdout) becomes the reason. Any other non-zero is logged and ignored. Exit `2` from the other events does nothing.

## Busy/idle pattern (verified 2.0.6)
- `UserPromptSubmit` hook sets busy (and doubles as proof the prompt arrived); `Stop` hook sets idle. Live runs recorded `busy` then `idle` for a normal turn.
- Write a small script that reads stdin, appends a record keyed by `session_id`, prints nothing, always exits `0`.
- Verified gaps: no `StopFailure` or `SessionEnd` event, so an auth error or crash leaves the record busy until the next prompt. `SessionStart` does not fire when the TUI starts. A single Escape interrupt ends the running tool but fires no `Stop`. Pair hooks with a pane or process check for stuck states.

## PreToolUse guard (docs)
Deny a call while auto-approve is on:
```json
{ "hookSpecificOutput": { "hookEventName": "PreToolUse",
  "permissionDecision": "deny", "permissionDecisionReason": "<why>" } }
```
printed on stdout with exit `0`, or just exit `2` with the reason on stderr. Match with `"matcher": "^execute_command$"` or `^write_file$`. `updatedInput` replaces the tool input but normal approval still applies. A hook runs with full user rights and no sandbox.

## Pitfalls
- Printing anything from a `UserPromptSubmit` or `SessionStart` hook injects it into the model context and pollutes the task. Redirect stdout to `/dev/null`.
- A bug that exits `2` in `UserPromptSubmit` blocks every prompt. Guard the script with `|| true` and exit `0`.
- Slow hooks: default timeout is 10 s and runs in the turn path; keep it near-instant.
- Workspace hooks in an untrusted folder are dropped silently; register trust first (`bob-approvals-and-trust`).
- A cloned repo's own `.bob/settings.json` hooks run too once the folder is trusted: review before trusting (`bob-security-and-data`).
- Persisting `prompt` or `tool_input` from the payload copies secrets into logs.
- Enterprise `EnforcedHooks` is a Bob IDE policy; do not expect it in Shell.

## See also
`bob-approvals-and-trust`, `bob-launch-interactive`, `bob-acp-integration`, `bob-troubleshooting-pitfalls`, `bob-config-rules-skills`.

## Sources
`shell/configuration/lifecycle-hooks`, `shell/security/trusted-folders`, `ide/security/group-policies`, live runs on Bob 2.0.6 (busy/idle, SessionStart gap, interrupt).
