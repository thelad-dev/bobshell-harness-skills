---
name: bob-sessions-resume
description: Load when you must continue, list, export or reset a Bob Shell task (--resume, --list-tasks, /resume, /clear, /export), or pick an instance or team (--team-id, --instance-id, /team) so an unattended Bob worker does not hit a picker.
---

# Bob tasks, resume, and instance/team selection

How a steering agent keeps hold of a Bob task across turns and relaunches, and avoids interactive pickers. Unmarked = docs; `(verified 2.0.6)` = local `bob --help` or a live run on 2.0.6.

## When to load
- A worker ended, was interrupted, or you relaunch and want the same context back.
- You need a task id, a transcript export, or a fresh task in a live pane.
- Bob asks for an instance/team or shows a picker you did not expect.

## Task ids
- Every run is a task with a UUID. `bob run -f json` returns it as `stats.task_id`; store it with your own task record (verified 2.0.6). This is the reliable id source for unattended work.
- Interactive tasks: `bob --list-tasks [n|all]` lists tasks of the current workspace (default 20). Docs say it switches to NDJSON when stdout is not a TTY: `{"id","title","status","workspace","updatedAt"}` (status `active|completed|paused`).
- Local 2.0.6 differs (verified): without a TTY, `bob --list-tasks </dev/null` fails with an invalid `--prompt` error ending in `Too small` (the root command reads stdin as a prompt; the message text follows the locale), and with stdin left open it hangs for minutes. Under a pty it prints a table with the columns ID, updated and title (labels follow the locale) (`script -qc "bob --list-tasks 20" /dev/null`). NDJSON output not seen on 2.0.6; try on 2.0.9+.
- Run list calls from the intended workspace and give them a timeout.
- Retention: 30 days by default; setting `tasks.retentionDays` to `0` keeps all (2.0.1 changelog).

## Resume
```
bob chat --resume <task-id> --auto-approve --trust --workspace /abs/ws --mode agent
bob run  --resume <task-id> -f json --max-turns 20 "Continue: fix the failing test." </dev/null
bob chat --resume latest ...        # most recent task
```
- `-r, --resume [task-id]` has an OPTIONAL value (verified 2.0.6 help). A bare `--resume` opens the picker and blocks an unattended pane. Always give an id (or `latest`) directly after the flag, before the prompt.
- Prefer an explicit id. `latest` is "most recent task" and can pick another worker's task when several run in parallel (inferred).
- Resume restores the conversation (docs), not launch settings (inferred): re-pass `--workspace`, `--mode`, caps, trust and the API key. Caps apply to the new process (inferred).
- Relaunch in the same pane: wait for the old composer to clear, then deliver the next brief file (see `bob-launch-interactive`).
- Tasks created through `bob acp` are shared with Bob Shell and connected editors (2.0.5 changelog), so a task started over ACP can be resumed from the CLI.

## Slash commands for task control (interactive)
| Command | Aliases | Use |
|---|---|---|
| `/resume` | `/history` | Opens a picker of past conversations. Interactive only; needs navigation keys, prefer `--resume <id>` at launch. |
| `/clear` | `/new`, `/task` | Start a fresh task in the same pane (drops context). |
| `/compact` | `/condense`, `/compress`, `/summarize` | Compact the context window when the footer shows high context use. |
| `/export <json\|markdown> [file]` | | Save the current task into the workspace; read it back as evidence. In the resume dialog `Ctrl+E` exports the highlighted task. |
| `/status` | `/info` | Session status, usage, version. |

## Instance and team (`/team`)
- A user can belong to several instances and, on enterprise plans, teams. `/team` opens a table (INSTANCE, PLAN, TEAM, BUDGET, USAGE). Arrows move, Enter selects, Esc cancels. Shown only with more than one team profile. The doc page is filed under "instance-command".
- The choice persists across sessions and refreshes authentication, so selecting it in one pane changes the default for the user's other Bob sessions.
- First-time setup forces an instance choice and disables Esc. An invalid `--team` or `--instance` argument also opens the mandatory picker; Ctrl+C exits it (2.0.0 changelog). This is a stall point for headless workers.
- Avoid the picker by authenticating with an API key and passing identifiers by flag (verified 2.0.6): `--team-id <TEAM_ID>` (required for `general`-type keys) and, on `chat`, `--instance-id <ID>`. Docs say `--instance-id` attaches to a named instance; local help says it preselects an instance for the session. Treat it as preselect.
- Do not store team or instance ids in skills, briefs or reports; keep them in the launcher's environment or config.
- `/team` and `/status` show budget and usage per instance or team. That is the only in-Bob quota view (see `bob-cost-and-limits`).

## Pitfalls
- Bare `--resume` or `/resume` in an unattended pane blocks on a picker; the worker looks idle forever.
- `--list-tasks` without a pty or with open stdin fails or hangs on 2.0.6; do not use it in a loop without a timeout.
- Resuming under another workspace or after a trust change may lose hooks or tool access (inferred); re-check the footer and workspace after launch.
- Task history keeps shell command output (2.0.3), so an exported task can contain secrets typed or printed earlier. Do not publish exports unchecked.
- A `latest` resume after a failed or empty run may attach to the wrong task; log the id you resumed.

## See also
`bob-launch-interactive`, `bob-run-headless`, `bob-cost-and-limits`, `bob-acp-integration`, `bob-security-and-data`.

## Sources
`shell/getting-started/start-bobshell-interactive`, `shell/getting-started/start-bobshell-non-interactive`, `shell/features/instance-command`, `shell/features/slash-commands`, `shell/configuration/keyboard-shortcuts`, `shell/changelog`, `local-help` (`bob --help`, `bob chat --help`, bob 2.0.6; `bob --list-tasks` run locally; `stats.task_id` and the stdin hang observed on 2.0.6).
