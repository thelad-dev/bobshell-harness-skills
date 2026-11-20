---
name: bob-brief-and-context
description: Load when writing the prompt or brief for a Bob Shell task, deciding which files and context to hand over, splitting work into Bob sessions, or recovering from a drifting or poisoned Bob session.
---

# Brief and context for Bob

How to write the brief, feed context via `@` mentions and `AGENTS.md`, cut tasks, and avoid context poisoning, so Bob's result is right the first time.

## When to load
- You are writing a prompt, brief file or follow-up for Bob.
- A task is large and you must split it or choose what Bob reads.
- Bob's output drifts, repeats, loops, or contradicts the repo.

## Brief skeleton
Docs prompt components: instruction, role, context, example, cue. For an agent-to-agent brief use this order:
```text
Goal: <one sentence, the observable outcome>
Inputs: @src/billing/invoice.ts @tests/invoice.test.ts   # exact paths, line ranges if known
Do: <numbered steps or the one change wanted>
Do not: <files, behaviors, refactors, dependencies off limits>
Pattern: follow @src/billing/credit.ts            # an example beats a description
Done when: `npm test -- invoice` passes; no other file changed
Report: end with 5 lines max: files changed, command run, result, open risks
Do not ask questions; state assumptions in the report.
```
- Be specific. "Fix the email validation in src/utils/validation.ts lines 45-67" works; "review everything in src/" does not (docs).
- Put the cue (report format) last; with `bob run --format json` the `last_message` field is that final message.
- A role line ("senior Python engineer") is optional; docs show it helps style. A done-condition and an example help more (opinion).
- State constraints and evidence the model cannot derive: the failing command, the exact error line. Trim logs to the relevant lines.
- Never put secrets, tokens or key values in a brief. Brief text lands in session history and logs.

## Delivering the brief
```bash
# Headless: prompt on stdin or as argument, workspace explicit
cat /abs/path/in/workspace/brief.md | bob run --workspace /abs/workspace --mode agent --max-cost <N> --max-turns <N>
```
- Interactive `bob chat` has no positional prompt (verified 2.0.6); type or send a short pointer: `read <workspace>/.bob/launch-brief.md and follow it exactly.`
- Keep the brief file inside the workspace. Bob blocks file reads outside it by default. Add it to `.git/info/exclude` so it is never committed.
- Multi-line prompts: save to a file and pipe it (docs).
- `bob -p/--prompt` exists at top level (verified 2.0.6 help); its behavior was not tested. Prefer `bob run`.

## `@` context mentions (docs)
- `@path/file.ts` inlines the file: up to 100 KB per file, 500 KB total across mentions, directories list up to 50 entries.
- `@dir/` for a whole directory is broad; name files instead.
- `@server:resource` reads an MCP resource of a connected server.
- Mention only files Bob needs for this step. Let Bob search for the rest; reading costs Messages tokens, not a separate line (docs).

## Standing context
- `AGENTS.md` is loaded at the start of every task: `~/.bob/AGENTS.md` (global), project root and parents, subdirectories (component scope). Mode rules: `.bob/rules-<mode>/` (docs).
- `/init` generates `AGENTS.md` and `.bob/rules-{agent,ask,plan}/AGENTS.md`. Re-run after structure changes; do not run it unasked in a repo you do not own (it writes files).
- Keep it short and operational: test, build, lint commands and style rules. It is re-sent with every prompt (docs). Do not repeat it in the brief.
- Global rules in `~/.bob/rules` also apply to every Bob task on this machine; know what is there before you blame the brief.

## Cutting tasks
- One session per work goal. New topic, new session (docs). A `bob run` is a new task each time; `/clear` starts one in chat.
- Stage it: find files, inspect, plan, change, validate (docs). As steering agent, make stages separate Bob tasks when they are large: survey in ask mode, spec in plan mode, edit in agent mode. Pass the result forward as a file or a short summary, not the whole transcript.
- Prefer small approved changes over one big autonomous run so diffs stay reviewable (docs).
- Large file: ask for an outline first, then target one function per task (docs).
- Broad repo reads: tell Bob to use an explore subagent and return only findings (`bob-modes-and-orchestration`). Bob decides whether to spawn; it is not guaranteed.
- Use git as the safety net: commit before a task; on a bad result `git revert`/reset and start a fresh session instead of correcting in place (docs).

## Context poisoning
Wrong or stale text stays in Messages and is re-sent on every prompt (docs). Signs: repeated suggestions, tool calls that do not match the ask, loops, fixes that last one turn.
- Reset: new session. A corrective prompt only masks it for one or two turns (docs).
- Paste less; check tool output for garbage and stop on it; split unrelated goals.
- When comments, README and code disagree, point Bob at a runnable check; trust tests over text (docs).
- Compaction starts near 190k of 270k tokens and is lossy; if a run gets that long, split the task instead (`bob-cost-and-limits`).

## Pitfalls
- Brief outside the workspace: Bob cannot read it and stalls or invents. Keep it inside.
- Naming `.bob/settings.json` in a brief or command: Bob asks for explicit approval even with `--auto-approve` (observed 2.1.0, `bob-approvals-and-trust`). `.bob/launch-brief.md` is fine.
- "Check the backend" style asks: Bob explores broadly, fills the window, and costs more.
- Resuming a poisoned task with `--resume`: the bad history comes back with it. Start fresh.
- Pasting command output with truncation or odd characters: becomes "fact" in the transcript.
- No done-condition: Bob stops when it thinks it is finished, not when it works.

## See also
`bob-modes-and-orchestration`, `bob-run-headless`, `bob-launch-interactive`, `bob-sessions-resume`, `bob-config-rules-skills`, `bob-cost-and-limits`, `bob-verify-output`.

## Sources
`shell/getting-started/best-practices`, `shell/tutorials/introduction`, `shell/tutorials/start-a-project`, `shell/tutorials/write-effective-prompts`, `shell/core-concepts/context-poisoning`, `shell/core-concepts/context-window-management`, `shell/core-concepts/large-projects`, `shell/getting-started/start-bobshell-interactive` (mention limits), `shell/getting-started/start-bobshell-non-interactive`; `local-help` (`bob --help`, 2.0.6).
