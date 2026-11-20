---
name: bob-launch-interactive
description: Load when you must start Bob Shell as an interactive worker (`bob chat`) in a terminal pane, hand it a brief, watch readiness, interrupt it, or exit it. Covers exact launch line, flags, trust, prompt delivery and keys.
---

# Launch Bob Shell interactively (`bob chat`)

How a steering agent starts, feeds, interrupts and ends a Bob TUI worker. Unmarked facts are from the docs; `(verified 2.0.6)` = local `bob --help` or a live run on Bob 2.0.6.

## When to load
- You run Bob as a long-lived worker in a tmux pane and need several turns, not one shot.
- You need the exact launch line, readiness check, prompt delivery, interrupt or exit.
- A Bob pane looks stuck (dialog, license, login, trust).

## Launch line
Use absolute paths; a pane daemon's PATH can differ from yours (verified 2.0.6).
```
env -u BOB_SESSION -u CLAUDECODE LC_ALL=en_US.UTF-8 \
  /abs/path/bob chat --auto-approve --trust --workspace /abs/worktree --mode agent
```
- `bob chat` has NO positional prompt (verified 2.0.6). Deliver the brief after the TUI is ready (see below).
- No `--model` or `--effort` flag (verified 2.0.6). Model choice is not steerable per launch.
- `env -u BOB_SESSION`: Bob sets `BOB_SESSION=1` for its tool subprocesses and throws on a nested session. Clear it when Bob launches Bob (verified 2.0.6).
- `LC_ALL`: Bob translates its UI from `LC_ALL`, `LC_MESSAGES`, then `LANG`. Under `de_DE` the placeholder and footer are German (verified 2.0.6). Pin English if you read the screen.
- Check `bob -v` first. A stale `bob` on PATH (1.0.6) rejects `chat --auto-approve --workspace` with "Unknown arguments" (verified). Support for 1.x and 2.0.8 and older ends in early 2027.

| Flag (verified 2.0.6) | Use |
|---|---|
| `--auto-approve` | Skip tool approval prompts. Silently ineffective in an untrusted folder; two dialogs still ask in a trusted one (`bob-approvals-and-trust`). |
| `--trust` | Mark the folder trusted (persists). Without trust, auto-approve and workspace hooks are suppressed. |
| `-w, --workspace <path>` | Workspace root. Bob blocks file reads outside it by default. |
| `--mode <slug>` | Start mode (`agent` default, `plan`, `ask`, custom slug). `/mode` takes no argument, so set it here. |
| `--max-cost <n>`, `--max-turns <n>` | Caps in Bobcoins and turns; `0` removes the cap. Docs list them only for `run`; `chat --help` has them too. |
| `--disable-mcp`, `--disable-subagents`, `--disable-tool-groups <list>` | Shrink the tool surface. |
| `--team-id <id>`, `--instance-id <id>` | See `bob-sessions-resume`. |
| `-r, --resume [task-id]` | See `bob-sessions-resume`. |
| `--accept-license` | Skip the license dialog. |
| `--log-level <lvl>` | `debug` ... `silent`, or env `BOB_LOG_LEVEL`. |

Docs also list `--config-dir`; 2.0.6 help does not (use 2.0.9+).

## Trust
- Pre-register the worktree so the first-run trust dialog never blocks: `~/.bob/trustedFolders.json` holds `{"version":1,"folders":{"/abs/worktree":"TRUST_FOLDER"}}` (verified 2.0.6). Do this AND pass `--trust`. Leave the entry after teardown; it is per path, not per spawn.
- Trust and approval policy details: `bob-approvals-and-trust`.

## Auth without a browser
- Set the API key in the environment of the Bob process only: `BOB_API_KEY` (docs; older name `BOBSHELL_API_KEY`). Never put it in argv, brief or logs; there is no key flag (verified 2.0.6).
- A launcher script that reads a 600 key file and `exec`s Bob keeps the key out of `ps` (verified 2.0.6).
- General-type keys also need `--team-id <TEAM_ID>`.
- Without a key Bob opens a browser SSO login. Do not let an unattended worker hit that path.

## Ready, then deliver the brief
1. Poll the pane until the composer row is idle: `❯` between two `─` rules with the grey placeholder `Build Anything, @ for context, / for commands, $ for skills` (verified 2.0.6). Footer shows mode, context, cost, and a red `(auto-approve)` suffix.
2. Dialogs block the composer: license, login, trust, team picker. With a fresh `HOME`, 2.1.0 shows the license dialog until top-level `licenseConsent` is set (verified). Do not type into a pane without an idle composer.
3. Put the full brief in a file INSIDE the workspace (e.g. `/abs/worktree/.bob/launch-brief.md`, git-excluded). Type one line and Enter: `read /abs/worktree/.bob/launch-brief.md and follow it exactly.` (verified 2.0.6).
4. Confirm delivery: your text sits above an empty composer, or a `UserPromptSubmit` hook fired (`bob-hooks`).

## Keys the steering agent can send
| Key / input | Effect |
|---|---|
| Enter | Submit. `Ctrl+J` inserts a newline, so long text goes through a file, not keystrokes. |
| Escape | Interrupt a streaming response or running tool call (verified 2.0.6: ends the running tool call). |
| `Tab` while streaming | Queue the message; queued items send in order after each turn. |
| `Ctrl+C` | Exit, close dialogs, reset input; may need two presses. |
| `Ctrl+Y` | Toggle auto-approve (2.0.6+; was Ctrl+F or Alt+F earlier). |
| `Shift+Tab` | Cycle modes. |
| `/exit` (`/quit`, `/close`) | Clean exit (verified 2.0.6). |

Other slash commands worth sending: `/compact` (free context), `/clear` (new task), `/export markdown <file>` (save the task into the workspace), `/status` (usage, version). Full list: `bob-tools-overview`.

## Pitfalls
- A first char `!` on an empty input toggles shell mode; your text is then taken as a shell command (docs: shell mode, leave with Escape or `!`). Never start a brief line with `!`.
- A line starting with `/` is treated as a slash command (inferred from docs); an absolute path as first token may be rejected as an unknown command. Start with a word.
- `Stop` hook does not fire on Escape or on auth errors, so a record can stay "busy" until the next prompt (verified 2.0.6).
- After a relaunch in the same pane the old composer stays on screen about 5 s; a stale pointer line looks like delivery. Check the cursor or a fresh hook record (verified).
- Foreground process of the pane is `node .../bob.js`, not `bob`; match on the script path when you look for it (verified 2.0.6).
- Closing the pane does not stop Bob; one SIGTERM does (verified 2.0.6). Interactive login shells ignore SIGTERM, so expect to reap the shell too.
- Default `bobShell.autoUpdate` runs `npm install -g --force` at start and can replace Bob under a running worker. Set it to `false` in `~/.bob/settings/settings.json` (user settings only; per-task pinning is impossible) (verified 2.0.6).
- If the project tracks its own `.bob/settings.json`, your hook and brief files there clobber it. Refuse or work around.
- A worker can stop mid-turn on "Command is too complex and needs manual verification" or "This operation accesses protected Bob files and requires explicit approval" despite `--auto-approve --trust` (observed 2.1.0). The turn does not end; read the pane and answer it, details in `bob-approvals-and-trust`.

## See also
`bob-run-headless`, `bob-sessions-resume`, `bob-approvals-and-trust`, `bob-hooks`, `bob-brief-and-context`, `bob-troubleshooting-pitfalls`.

## Sources
`shell/getting-started/start-bobshell-interactive`, `shell/getting-started/install-and-setup`, `shell/configuration/keyboard-shortcuts`, `shell/features/slash-commands`, `shell/changelog`, `shell` (index), `local-help` (`bob --help`, `bob chat --help`, bob 2.0.6).
