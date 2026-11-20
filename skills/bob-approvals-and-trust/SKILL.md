---
name: bob-approvals-and-trust
description: Load when deciding how much autonomy to give Bob Shell (auto-approve, approval groups, allow/deny commands) or when Bob seems to ignore --auto-approve, hooks, skills, MCP or AGENTS.md because a folder is untrusted.
---

# Bob approvals and trusted folders

How a steering agent grants Bob exactly the autonomy it needs, and the trust switch that silently turns most of it off.

## When to load
- Choosing between prompting, `--auto-approve` and settings-based approval for a Bob run.
- Bob stops on approval prompts nobody can answer, or runs without the autonomy you asked for.
- Hooks, project skills, `.bob/mcp.json`, `AGENTS.md` or `--auto-approve` have no effect in a worktree.
- Registering a fresh worktree or CI checkout with Bob.

## Approval layers (docs, flags verified 2.0.6)
1. Default: every tool call prompts, one at a time. Commands can be edited before approval.
2. `bob chat --auto-approve` and `bob acp --auto-approve`: "auto-approve tool executions when security rules allow" (verified 2.0.6). It is conditional, not blanket: denied commands and untrusted folders still prompt, and two dialogs remain even in a trusted folder (see "Prompts that remain under auto-approve").
3. `~/.bob/settings/settings.json`, key `approval` (read at session start, so edit before launching):
```json
"approval": {
  "allowed_permissions": ["read", "todo", "subtask", "subagent", "mode"],
  "permissionOptions": [{ "groupId": "read", "enableOutsideWorkspace": true }],
  "allowedExecutors": [{ "toolId": "execute_command",
    "approvedCommands": ["git status", "git diff", "ls", "grep"],
    "deniedCommands": ["rm"] }]
}
```
4. In a session: `/permissions` toggles auto-approve and changes folder trust; task-level overrides last for the current task only.
5. MCP tools: per-server `alwaysAllow` in `mcp.json` (see `bob-mcp-setup`).

`bob run` (2.0.6) has no `--auto-approve` flag (verified; the trust docs mention one for run, local help does not). The non-interactive docs say all tools are pre-approved in `bob run`, so there is no approval gate there: restrict with `--mode`, `--disable-tool-groups`, trust and a clean git checkout. Settings-based approval (layers 3 and 5) matters for `bob chat` and `bob acp`. Behaviour of a call that would need approval in `bob run` was not tested.

## Prompts that remain under auto-approve
`bob chat --auto-approve --trust` in a trusted folder still stops on two dialogs (observed live on 2.1.0 in a Firstmate adapter run; message texts and the `isCommandSecurityEnabled` default found in the 2.1.0 package). There is no stronger flag; the steering agent must read the pane and answer the dialog (the Firstmate adapter sends Enter after reading the pane).
- "Command is too complex and needs manual verification": Bob's command security check. It is on by default for `bob chat` (top-level `isCommandSecurityEnabled`, default `true`) and can be switched off only in the user settings `~/.bob/settings/settings.json`; a workspace `.bob/settings.json` does not reach it, so it cannot be turned off per task. `bob run` skips the check unless `--command-security` is passed.
- "This operation accesses protected Bob files and requires explicit approval" (Approve Once or Reject): built in, cannot be switched off. It fires when a command names `.bob/settings.json`, even only to read it, or when the edit tool changes a file under `~/.bob/`. Keep that path out of briefs; a brief file such as `.bob/launch-brief.md` is not protected.

## Groups and what they allow
| Group | Covers | Risk |
|---|---|---|
| `read` | read, list, search | medium |
| `edit` | write, create, delete files | high |
| `execute` | any shell command | high |
| `mcp` | all calls of all configured MCP servers | medium-high |
| `skill` | activating skills | medium |
| `todo`, `subtask`, `subagent`, `mode` | workflow only | low |

- `approvedCommands` match as a prefix of the full command string. `deniedCommands` always win over auto-approve; on overlap the longer match wins, a tie allows. Docs do not say how chained commands (`a && b`) are matched: keep entries narrow and do not rely on denial of `rm` to stop `sh -c`.
- `read` is limited to the workspace. `enableOutsideWorkspace: true` lifts that. Without it, put briefs and inputs inside the worktree (a launcher can copy the brief to `.bob/launch-brief.md` for this reason).

## Autonomy ladder
- Research or review: modes plan/ask, no auto-approve, or only `read`, `todo`, `subtask`, `subagent`, `mode`.
- Implementation in a disposable worktree: `bob chat --auto-approve --trust --workspace <abs-worktree>` plus `.bobignore` for secrets (`bob-security-and-data`).
- Never auto-approve `edit`/`execute` in a checkout with production credentials or a shared working tree.
- Confirm it took effect: the German-locale footer reads `Agent Mode (automatische Genehmigung)` when auto-approve is active (verified 2.0.6). Bob translates its UI from `LC_ALL`, `LC_MESSAGES`, `LANG`.

## Trusted folders
- Docs: the feature is off by default and then every folder counts as trusted. Enable with `/settings` (Folder Trust) or `security.folderTrust.enabled: true`; restart needed. treat trust as always required in automation and register the folder anyway.
- Store: `~/.bob/trustedFolders.json`. Shape seen locally: `{"version":1,"folders":{"<abs-path>":"TRUST_FOLDER"}}` (verified 2.0.6). Docs show a flat path map: prefer the versioned form. Levels: `TRUST_FOLDER` (exact path), `TRUST_PARENT` (path and all below), `DONT_TRUST`.
- Register a worktree: `bob chat --trust --workspace <abs-path>` writes the entry (docs). `bob run --trust` trusts for that run only and persists nothing (docs). `bob acp --trust` trusts every workspace that server opens (verified 2.0.6).
- Entries survive worktree teardown (verified 2.0.6). Concurrent writers can overwrite each other: serialize edits and write the file atomically.
- Interactive first visit shows a trust dialog nobody answers headlessly. `bob run` never asks: unresolved folders count as trusted, `DONT_TRUST` aborts with "is not a trusted folder. Pass --trust".

## What an untrusted folder silently disables (docs, verified 2.0.6)
`.bob/settings.json` (hooks, project settings), `--auto-approve` (every tool prompts), MCP servers (no connection), project custom modes, project skills, custom subagents, and `AGENTS.md`/project rules. Symptoms: a busy/idle hook never fires, `$skill` not found, the brief in `AGENTS.md` ignored, tools stall on prompts. First check trust, then everything else.

## Pitfalls
- Assuming `--auto-approve` is enough: in an untrusted folder it is dropped without any message and the run hangs on a prompt. Even when trusted, the two dialogs above can stop a worker mid-turn, with no turn end.
- Adding `execute` to `allowed_permissions` globally affects every later Bob session on the machine, not only yours.
- Policies such as `DisabledAutoApprovalGroups` are Bob IDE only; do not expect them to cap Shell.
- `forbiddenApprovalGroups` appears under `approval` in a local settings file but is undocumented: unverified, do not depend on it.

## See also
`bob-security-and-data`, `bob-hooks`, `bob-run-headless`, `bob-launch-interactive`, `bob-config-rules-skills`, `bob-mcp-setup`.

## Sources
`shell/configuration/approval-settings`, `shell/features/auto-approving-actions`, `shell/security/trusted-folders`, `shell/security/bob-security-guidance`, `ide/security/group-policies`, `local-help` (`bob chat|run|acp --help`, Bob 2.0.6); remaining dialogs: the Bob adapter reference in Firstmate (`harness-adapters/references/harness/bob.md`, Bob 2.1.0) and the `@ibm/bob` 2.1.0 npm package.
