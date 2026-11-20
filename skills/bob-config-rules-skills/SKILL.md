---
name: bob-config-rules-skills
description: Load when you need to know where Bob Shell reads settings, AGENTS.md, rules and skills from, which scope wins, or how to install a skill folder or standing rules for Bob.
---

# Bob config, rules and skills

Where Bob's configuration lives, what takes precedence, and how to give Bob standing instructions or an installed skill.

## When to load
- Giving Bob standing project instructions instead of repeating them in every brief.
- Installing a SKILL.md folder (for example one of these `bob-*` skills) so Bob can load it.
- A setting, rule or skill seems ignored, or two scopes conflict.
- Changing defaults (mode, max turns, retention, telemetry, MCP, subagents).

## Locations
| What | User scope | Project scope |
|---|---|---|
| Settings (`approval`, `hooks`, `session`, ...) | `~/.bob/settings/settings.json` | `.bob/settings.json` |
| MCP servers | `~/.bob/settings/mcp.json` | `.bob/mcp.json` |
| Rules | `~/.bob/rules/` | `.bob/rules/`, `.bob/rules-<mode>/` |
| Context file | `~/.bob/AGENTS.md` | `AGENTS.md` in root, parents and subdirs |
| Skills | `~/.bob/skills/<name>/SKILL.md` | `.bob/skills/<name>/SKILL.md` |
| Custom modes | `~/.bob/settings/custom_modes.yaml` (file seen locally, verified 2.0.6) | project folder (docs) |
| Logs | `~/.bob/logs/shell/` | |

- Precedence (docs): CLI flags override files for that session. A project skill or MCP server beats a global one of the same name. For plain settings keys the docs do not state project-vs-user order: do not rely on project overriding user for them.
- Settings are read when a session starts; restart Bob after edits. `/settings` edits interactively. Everything project-scoped loads only in trusted folders (`bob-approvals-and-trust`).
- Useful keys: `session.maxTurns` (100, `0` unlimited), `session.defaultMode` (`agent`), `session.mcp`, `session.subagents`, `tasks.retentionDays` (30; unpinned tasks older than that are deleted, which removes them from `--resume`; `0` keeps all, only on 2.0.1+), `telemetry.enabled`, `logging.logLevel`.
- Auto-update: `bobShell.autoUpdate` runs `npm install -g --force` and is read from user settings only: a launcher cannot pin it per task.
- Also on disk locally: legacy `~/.bob/settings.json` and `~/.bob/settings/mcp_settings.json`. Not documented for 2.0.6; `bob mcp list` shows nothing from them (verified). Use only the paths in the table.
- Verbose logging: `BOB_LOG_LEVEL=debug bob chat` or `--log-level debug`; `/logs` opens the latest log.

## Rules and AGENTS.md
- `AGENTS.md` is loaded automatically as workspace rules for every mode (turn off with `"bob-shell.useAgentRules": false`). A repo's `AGENTS.md` written for another agent will steer Bob too: read it before the run.
- Directories: `rules/` (all modes), `rules-agent/`, `rules-plan/`, `rules-ask/`, `rules-{mode-slug}/` for custom modes. Files are read recursively in alphabetical order; `.DS_Store`, `*.bak`, `*.cache`, `*.log`, `*.tmp` skipped; symlinks to depth 5; empty files ignored.
- Order: global before workspace, mode-specific before general; workspace overrides global. `AGENTS.md` loads after mode-specific rules but before general workspace rules.
- Put stable constraints (style, commands to run, do-not-touch paths) in `.bob/rules/*.md` or `AGENTS.md`; put the task in the brief. In an untrusted folder none of these load.
- Bob cannot read outside its workspace by default: place per-task briefs inside the worktree (for example `.bob/launch-brief.md`), see `bob-brief-and-context`.

## Skills: how Bob finds and uses them
- Discovery: `.bob/skills/<name>/SKILL.md` (project) and `~/.bob/skills/<name>/SKILL.md` (global). Folder = lowercase kebab-case, up to 64 chars, equal to `name`. Bad folder names are skipped without any error.
- SKILL.md needs front matter `name` and `description`. Docs say a missing description falls back to the first line; do not rely on it (other sources say such skills are ignored), always write one. Keep to those two keys.
- Use: type `$` for the picker, `$<name> task text` to force a skill, or `/skills`. Without naming it, Bob matches the request against the description. Skills load once per conversation; supporting files in the folder (`references/`, scripts) are read after activation.
- Permission: activation asks for approval unless `"skill"` is in `approval.allowed_permissions`; add it for unattended runs.
- Install a skill folder for Bob:
```
mkdir -p ~/.bob/skills && cp -R <abs-skill-folder> ~/.bob/skills/<name>
# per task: <worktree>/.bob/skills/<name>, keep out of git via .git/info/exclude
```
Restart Bob, then check that `$` lists it. Whether Bob follows symlinks in `~/.bob/skills` was not tested, copy when in doubt. Check with `ls ~/.bob/skills/<name>/SKILL.md`.
- Same name in both places: project wins. Untrusted folder: project skills not loaded, global ones still are.

## Pitfalls
- Skill not listed: wrong folder name (uppercase, underscores), missing `description`, untrusted folder, or Bob not restarted.
- Editing `~/.bob/settings/settings.json` while a session runs: no effect until restart.
- Project `AGENTS.md` aimed at Claude or Codex conventions silently applies to Bob and can conflict with your brief.
- Setting `tasks.retentionDays` to `0` on Bob older than 2.0.1 deletes unpinned task history.
- Stale `bob` in the login `PATH` (an old `bobshell` 1.0.x, seen on one host) fails on current flags such as `chat --auto-approve --workspace`; check `bob --version`.

## See also
`bob-approvals-and-trust`, `bob-brief-and-context`, `bob-modes-and-orchestration`, `bob-mcp-setup`, `bob-sessions-resume`.

## Sources
`shell/configuration/configuring`, `shell/configuration/bobshell-custom-rules`, `shell/features/skills`, `shell/tutorials/use-skills`, `shell/configuration/approval-settings`, `local-help` (`bob --version`, `bob mcp list`, `~/.bob` layout).
