---
name: bob-modes-and-orchestration
description: Load when choosing the Bob Shell mode for a task (agent, plan, ask, custom), restricting what Bob may do, sequencing multi-step work across Bob sessions, or deciding about subagents and background processes.
---

# Modes and orchestration

Pick the right Bob mode per task, define custom modes, and use Bob's subagents and background processes. The steering agent is the orchestrator; Bob has no orchestrator mode.

## When to load
- You set `--mode` or call `/mode` and want the right one.
- You need a read-only or tool-restricted Bob.
- A job has several stages (survey, design, implement, review).
- Bob should explore a lot of code or run long commands.

## Built-in modes (Bob Shell 2.x)
| Mode | Tools | Subagents allowed | Use for |
|---|---|---|---|
| `agent` (default) | read, edit, execute, mcp, skill, todo, subagent, mode (browser only with `experiments.webFetch`) | all | implement, fix, run commands |
| `plan` | read, edit on Markdown/text only, mcp, skill, subagent, mode | `explore` | design, spec, steps before coding |
| `ask` | read, mcp, skill, subagent (no edit, no execute, no mode switch) | `explore` | explain, survey, review without changes |

- `--mode` default is `agent` (verified 2.0.6). It takes a built-in name or a custom slug, on both `bob chat` and `bob run`.
- Docs and changelog: the old Code and Advanced modes were merged into Agent in 2.0.0. There is no orchestrator mode in the Shell docs. If the ibm-bob MCP bridge or an older prompt names `advanced`, `code` or `orchestrator`, treat them as unverified for 2.x; use `agent`. How the bridge maps these names was not tested.
- Plan mode loads a `create-plan` skill first and expects you to review the plan, then switch mode (docs).
- Premium IBM i modes (IBM i Developer, IBM i Database) are documented for the IDE package; whether `bob --mode` in the Shell offers them was not verified. See `bob-for-i-package`.

## Picking a mode per task
- Understand, locate, review, answer a question: `ask`. Safest and cheapest; no writes by construction.
- New feature or unclear approach: `plan` first. Have it write the plan to a Markdown file, read it yourself, then start a fresh `agent` task with `@plan.md` as input.
- Well-specified change with a test: `agent`.
- Review of an agent's diff: a new `ask` or `plan` task on the diff, not the same session that wrote it (opinion, matches the independent-review habit).
- Docs say Bob may switch modes by itself mid-task (for example plan to agent). In `bob run` that can turn a plan task into an implementation. If you want planning only, use `ask`, or keep plan and add `--disable-tool-groups mode` (inferred from the tool-group table, not tested).
- Interactive switch: `/mode` opens a picker, `/mode ask` switches directly.

## Restricting a run without a custom mode
```bash
bob run --mode ask "Summarize @src/"                                         # read-only by mode
bob run --disable-tool-groups edit,execute,mcp "Audit @src/ for SQL injection"   # read-only agent
bob run --disable-subagents --disable-mcp "Explain @app.js"
```
Groups: `read edit execute browser mcp skill todo subagent mode` (verified 2.0.6 help for the flag; group list from docs).

## Custom modes
Define in `~/.bob/settings/custom_modes.yaml` (user) or `.bob/custom_modes.yaml` (project), YAML preferred (docs).
```yaml
customModes:
  - slug: safe-reviewer
    name: Safe Reviewer
    roleDefinition: You review code for defects and report findings with file and line.
    whenToUse: Read-only reviews.
    customInstructions: |-
      Report findings only. Never propose edits as applied changes.
    groups:
      - read
```
- Edit limited to files: `- - edit` then `- fileRegex: \.(md|txt)$`. Omit `execute` to forbid commands.
- Select with `--mode safe-reviewer`. Mode-specific rules go in `.bob/rules-<slug>/`.
- Slug clash: built-in beats user beats project. Use unique slugs. A custom mode is a way to give Bob a hard capability boundary a brief cannot (opinion: groups are enforced by tool availability, a prompt is only a request).

## Subagents (docs)
- Isolated context window, returns only a summary to the main session. Types: `explore` (read-only, lighter model) and `general` (all tools, default model).
- Default is to work directly. Bob considers one only when the task is self-contained, would add a lot of irrelevant context, and needs more than one or two tool calls.
- Does not see the parent conversation unless Bob sets `fork_context: true`. In interactive chat you approve the spawn; `bob run` pre-approves.
- Use it by asking: "Survey @src/ with an explore subagent and give me only the list of relevant files and one line each." Bob still decides; do not count on it.
- Subtasks (`start_subtask`) are visible, interactive threads; subagents are silent. For headless runs prefer subagents.
- Turn off with `--disable-subagents` or `--disable-tool-groups subagent` when you want a single linear context or the lowest cost.

## Background processes (docs)
- Ask "run `npm run build` in the background" or let Bob background slow commands itself. Control returns immediately; `/processes` lists them in chat.
- Headless caveat (opinion, untested): a backgrounded command can outlive the turn. In a brief write "run tests in the foreground, wait for exit, report the exit code".
- Dev servers or watchers you start stay running; tell Bob to stop them before it reports done.

## Staging pattern
1. `ask`: survey, output a file list and facts (small summary back to you).
2. `plan`: write `plan-<topic>.md` in the workspace.
3. You review the plan.
4. `agent`: implement from `@plan-<topic>.md`, capped by `--max-cost` and `--max-turns`.
5. You run the tests; optionally a fresh `ask` task reviews the diff.

## Pitfalls
- `--mode` only picks the starting mode; Bob may switch. Do not rely on mode alone for safety, use `--disable-tool-groups` or a custom mode.
- Plan mode edits only Markdown/text. Asking it to change code gets a plan or a mode-switch request instead.
- Subagents hide tool steps: you see a summary only, so ask for evidence (paths, line numbers) in it.
- Custom mode with `execute` still needs approval rules; see `bob-approvals-and-trust`.

## See also
`bob-brief-and-context`, `bob-tools-overview`, `bob-run-headless`, `bob-config-rules-skills`, `bob-approvals-and-trust`, `bob-for-i-package`, `bob-cost-and-limits`.

## Sources
`shell/features/modes`, `shell/configuration/custom-modes-bobshell`, `shell/features/subagents`, `shell/features/background-processes`, `shell/getting-started/best-practices`, `shell/core-concepts/tools`, `shell/core-concepts/context-poisoning`, `shell/changelog`, `ide/premium-packages/bob-for-i/modes`; `local-help` (`bob run --help`, `bob chat --help`, 2.0.6).
