# bobshell-harness-skills

Agent skills for steering IBM Bob Shell from another coding agent.

This is a collection of 21 skills that teach an agent (Claude Code, Codex, Cursor, Grok or any tool that reads the Agent Skills format) how to drive IBM Bob Shell as a tool or worker: starting and briefing it (`bob chat`, `bob run`, `bob acp`), modes and commands, cutting tasks and context, approvals, trust and security, hooks, MCP setup, resuming sessions, checking its output, common pitfalls, and when Bob is the right choice at all. Two optional skills cover IBM i; the rest of the set has no IBM i dependency.

It is an independent, unofficial collection. It does not come from IBM and is not endorsed by IBM. The skills were built from the public IBM Bob Shell documentation (read on 2026-11-18, pages listed in [SOURCES.md](SOURCES.md)) and checked against Bob 2.0.6. Each statement is either from the docs (no marker) or observed on a local install (marked `(verified 2.0.6)`).

## Who it is for

Anyone who lets one coding agent delegate work to Bob Shell and wants the delegating agent to know the flags, traps and verification steps up front, instead of finding them by trial and error.

## Contents

```
bobshell-harness-skills/
  README.md        this file
  SKILLS.md        per skill: when it applies, which doc pages it was built from
  SOURCES.md       every page read, with address and date
  skills/<name>/SKILL.md   one folder per skill (only bob-troubleshooting-pitfalls also has references/versions.md)
```

Start with `bob-shell-steering`. It routes to the other skills and lists ten core rules. The skills are short, written in English, and contain no credentials, tokens, instance or team IDs.

## Format

Each skill is a folder with a `SKILL.md`. The header (YAML) has exactly two fields:

```
---
name: bob-xxx          # same as the folder name, lowercase, hyphens, at most 64 characters
description: One sentence on when the skill should be loaded.
---
```

This is the common Agent Skills format. Claude Code, Codex, Cursor, Grok and Bob read it unchanged. Bob requires `name` and `description` and skips folders with an invalid name.

## Installation

Copy whole folders (not just `SKILL.md`) and keep the folder names. Create the target folder if it does not exist, then restart the agent session.

```bash
git clone https://github.com/thelad-dev/bobshell-harness-skills.git
SRC="$PWD/bobshell-harness-skills/skills"
DEST=<target folder from the table>
mkdir -p "$DEST" && cp -R "$SRC"/. "$DEST"/
```

| Agent | User folder (all projects) | Project folder | Manual invocation | Note |
|---|---|---|---|---|
| Bob Shell | `~/.bob/skills/` | `<workspace>/.bob/skills/` | `$bob-...` or `/skills` | Discovery and loading tested (see below). Use only in a trusted folder; Bob asks before activating a skill unless `"skill"` is in `approval.allowed_permissions` or the run is `bob run`. |
| Claude Code | `~/.claude/skills/` | `.claude/skills/` | `/bob-shell-steering` | Claude lists all names but descriptions only within a budget (1 % of the context). With a tight context, set `skillOverrides` to `name-only` for everything except `bob-shell-steering`. |
| Grok | `~/.grok/skills/` | `.agents/skills/` | `/bob-shell-steering` | Paths not tested here. |
| Codex | `~/.agents/skills/` (or `~/.codex/skills/`) | `.agents/skills/` | `$bob-shell-steering` | Codex invokes skills with `$`, not `/`. Which of the two user folders your Codex reads was not tested here. |
| Cursor | `~/.cursor/skills/` | `.cursor/skills/` | `/bob-shell-steering` | Paths not tested here. |

Only Bob's own discovery of the skills was tested (below). For the other agents, check once after copying that the agent lists the skills.

A symlink instead of a copy works for Claude Code, Grok, Cursor and Codex. For Bob, please copy (whether Bob follows symlinks in `~/.bob/skills` was not tested).

### Installing only part of the set

Small contexts save descriptions: always install `bob-shell-steering`, and add others as needed (see the selection table below). Without IBM i, leave out `bob-ibmi-tools` and `bob-ibmi-ozgar-mcp` (see below). Only `bob-ibmi-ozgar-mcp` also belongs in the skill folder of the Bob worker itself (`<worktree>/.bob/skills/`); all other skills describe how to steer Bob and do not help Bob.

## Recommended IBM i tools (optional)

If you work on IBM i, I recommend these tools. Bob Shell gets the IBM i Premium Package only in the IDE, so these are the practical way for a Shell worker to read from an IBM i system. None of them is needed for the rest of the set.

| Tool | What it is for |
|---|---|
| OzgarAI (Ozgar MCP) | MCP server with a queryable knowledge graph of one IBM i project (artifacts, relationships, documentation, source), for structure and business logic; access needs an API key. Skill: `bob-ibmi-ozgar-mcp`. |
| [ibmi-axi](https://github.com/thelad-dev/ibmi-axi) | Read-only command line tool for live IBM i reads (objects, members, job logs, spool, messages, IFS), over SSH and for part of the commands also via ibmi-mcp; Bob calls it with `execute_command` in mode `agent`. Skill: `bob-ibmi-tools`. |
| [ibmi-mcp](https://github.com/IBM/ibmi-mcp-server) | IBM's MCP server for IBM i, giving an agent access tools to a live system (use a read-only profile); Bob loads it as `mcp__<server>__<tool>`. Skills: `bob-ibmi-tools`, `bob-mcp-setup`. |
| Mapepire | The database connection that ibmi-mcp uses to reach Db2 for i on the IBM i side; not covered further in these skills. |

`bob-ibmi-tools` is the general route: it checks what is installed (`command -v ibmi-axi` and `ibmi-axi doctor`, then the MCP tool list in Bob's session) and says "no IBM i access" instead of guessing when nothing is there. Ozgar MCP is one recommended option on top, for application structure.

## Which skills to load for which job (suggestion)

| Job for Bob | Skills to load |
|---|---|
| always | `bob-shell-steering` |
| Start a Bob pane as a worker | + `bob-launch-interactive`, `bob-approvals-and-trust`, `bob-hooks`, `bob-brief-and-context` |
| One-shot call or CI | + `bob-run-headless`, `bob-cost-and-limits` |
| Read-only work (survey, review) | + `bob-modes-and-orchestration`, `bob-verify-output` |
| IBM i question (only with IBM i tooling) | + `bob-ibmi-tools`, `bob-mcp-setup`; with Ozgar MCP also `bob-ibmi-ozgar-mcp` |
| Programmatic control | + `bob-acp-integration`, `bob-sessions-resume` |
| Trouble | + `bob-troubleshooting-pitfalls` |

## Checks done (Bob 2.0.6)

The 20 original folders were copied to `<workspace>/.bob/skills/` of a test checkout and checked in three Bob sessions:

1. `bob run -f stream-json`: Bob listed all `bob-*` names, loaded `bob-run-headless` through the `use_skill` tool without asking, and repeated two facts from the skill correctly.
2. `bob run`: Bob loaded `bob-shell-steering` and named exactly the skills it prescribes for "pane, worker, IBM i question".
3. Interactive `bob chat` (tmux): typing `$bob-ver` showed `$bob-verify-output` in the skill picker with its description and location `local`.

In addition, a project MCP server (`.bob/mcp.json`) was loaded and called as `mcp__<server>__<tool>`. `bob-ibmi-tools` was added after these checks and has not been run in Bob yet.

## Limits

- The skills rely on Bob 2.0.x. The docs partly require 2.0.9 or newer (for example `--command-security`, `--config-dir`); Bob 2.0.6 does not know these options, and the skills say so where it matters.
- The answers of the FAQ and "Common questions" accordions are missing from the source (only the questions are there). The model Bob picks is therefore not stated in any skill.
- `bob-for-i-package` and `bob-devsecops-package` record that both packages are, per the docs, described for the IDE only. For Shell workers on IBM i the fallback is `bob-ibmi-tools` (and `bob-ibmi-ozgar-mcp` with Ozgar MCP).
- Evidence markers such as `(verified 2.0.6)` mean the statement was observed on a local Bob 2.0.6 install, not taken from the docs.
- Re-read after every Bob upgrade: `bob-troubleshooting-pitfalls/references/versions.md` lists what to re-check.
