---
name: bob-tools-overview
description: Load to know what Bob Shell can do on its own (read, edit, execute, MCP, subagents, todo, mode switch), which tool groups each mode enables, and how to restrict or slim them for a task.
---

# Bob tools overview

What tools Bob has, which mode enables which, and how to restrict them so a task cannot do more than you intend.

## When to load
- You want to know whether Bob can do X (run commands, edit files, call MCP, fetch web pages).
- You need a read-only, no-execute or no-MCP Bob run.
- You want to cut token overhead from tool definitions.

## Tool categories (docs)
| Category | Tools | Notes |
|---|---|---|
| Read | `read_file`, `grep`, `glob`, `list_files` | free of side effects |
| Write | `write_file`, `apply_diff`, `insert_content`, `search_and_replace` | `apply_diff` is the targeted edit |
| Command | `execute_command` | runs in the workspace; slow commands may go to the background |
| MCP | `mcp__<server>__<tool>` | one tool per MCP server tool, e.g. from ibmi-mcp or Ozgar MCP |
| Mode | `switch_mode` | Bob changes mode on its own when work evolves |
| Question | `ask_followup_question` | asks the human for a choice or missing detail |
| Skill | `use_skill` | loads a skill's instructions (Plan mode starts with `create-plan`) |
| Todo | `update_todo_list` | agent mode only |
| Subagent | `spawn_subagent`, `start_subtask` | isolated context; see `bob-modes-and-orchestration` |
Also seen in a live 2.0.6 tool list (not on the docs tools page): `office_read`, `office_edit`, `list_ibm_doc_libraries`, `search_ibm_docs`. Treat them as available but undocumented; do not build on them.
Workflow: Bob picks a tool, shows parameters, runs it once approved, loops until done (docs).

## Tool groups per mode (docs)
| Group | Agent | Plan | Ask |
|---|---|---|---|
| `read` | yes | yes | yes |
| `edit` | yes | Markdown and text only | no |
| `execute` | yes | no | no |
| `browser` (web fetch, off unless `experiments.webFetch`) | yes | yes | yes |
| `mcp` | yes | yes | yes |
| `skill` | yes | yes | yes |
| `todo` | yes | no | no |
| `subagent` | yes | yes | yes |
| `mode` | yes | yes | no |

## Restricting tools
```bash
bob run --disable-tool-groups edit,execute,mcp "Summarize the codebase"   # read-only
bob run --disable-tool-groups execute "Draft docs for @src/api/"          # edits, no shell
bob run --disable-mcp "Explain @app.js"                                    # no MCP servers at all
bob run --disable-subagents "Fix @a.py"                                    # no subagent tools
```
- The same flags exist on `bob chat`, and `bob acp` has `--disable-mcp` and `--disable-subagents` (verified 2.0.6 help).
- `--disable-tool-groups` takes one group or a comma-separated list; everything else stays on (verified 2.0.6).
- A custom mode can fix a group set permanently (`bob-modes-and-orchestration`). `.bobignore` hides files from Bob (`bob-config-rules-skills`). Approvals, trust and command security decide what still runs: `bob-approvals-and-trust`.
- Docs list `--command-security` and `--config-dir` for `bob run`; they are not in the 2.0.6 `bob run --help`. Prefer local help: do not rely on them with 2.0.6.

## Token cost of tools
- Fixed overhead is re-sent with every prompt: for a trivial prompt about 8,500 tokens, of which about 5,100 are tool definitions (docs, one measurement).
- Each connected MCP server adds its tool definitions and instructions. Disable servers a task does not need, prefer project-scoped MCP config, or run with `--disable-mcp` (docs).
- Tool results and command output count toward Messages. Ask Bob to filter (`| tail -50`, `grep`, specific test names) rather than dump whole logs.

## What this means for briefs
- Say which tools matter: "use the ibmi-mcp read tools for object lookup, do not guess names" (see `bob-ibmi-tools`).
- Say "do not ask follow-up questions; state assumptions". An unattended run or worker pane that gets a question may stall (interactive chat shows it in the TUI; headless behavior untested).
- Plan mode cannot edit code, so a plan task writes only Markdown. Ask mode cannot write at all, not even a report file; take the answer from stdout, or use plan for a file.
- Web fetch is off by default and experimental; do not plan on Bob browsing.
- Bob runs commands in the workspace shell with the process environment, including any API key you exported. Do not export secrets Bob does not need.

## Pitfalls
- `--disable-tool-groups` with an unknown group name: not tested here, check the exit code and first output line before trusting a read-only claim.
- `execute` disabled does not stop MCP tools that run commands remotely. Disable `mcp` too for a hard read-only run.
- Switching modes mid-task changes available tools; a read-only plan can become writable. Disable the `mode` group if that matters.
- Many MCP tools or loaded skills inflate overhead before the first word; slim runs are cheaper and less confused.

## See also
`bob-modes-and-orchestration`, `bob-approvals-and-trust`, `bob-mcp-setup`, `bob-ibmi-tools`, `bob-hooks`, `bob-cost-and-limits`, `bob-config-rules-skills`.

## Sources
`shell/core-concepts/tools`, `shell/features/modes`, `shell/configuration/custom-modes-bobshell`, `shell/core-concepts/context-window-management`, `shell/features/subagents`, `shell/features/background-processes`, `shell/getting-started/start-bobshell-non-interactive`; `local-help` (`bob run --help`, `bob chat --help`, `bob acp --help`, 2.0.6).
