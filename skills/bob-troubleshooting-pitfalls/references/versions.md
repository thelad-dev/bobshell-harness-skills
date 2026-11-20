# Bob Shell version differences that matter to a steering agent

Open this when behaviour differs between hosts or after an upgrade. Marks: `docs` (changelog) or `(verified <version>)` (local CLI).

## Package and install timeline
| Version | Package | Notes |
|---|---|---|
| 1.0.x | `bobshell` | No skills, Code and Advanced modes, no `bob mcp` commands. `bob chat --auto-approve --workspace` is rejected ("Unknown arguments", verified on a 1.0.6 host). |
| 2.0.0 | `bobshell` | Settings moved to `~/.bob/settings/settings.json`. Agent mode replaces Code and Advanced. `bob run` json output gained `stats` and typed events. `bob mcp add/remove/list`. Skills added. |
| 2.0.1 | `bobshell` | 30-day task retention (`tasks.retentionDays`, 0 keeps all). Settings writes always need approval. `.git` beats `.bob` when resolving the workspace root. `--list-sessions` and `--limit` restored as silent aliases of `--list-tasks`. |
| 2.0.2 | `bobshell` | `bob acp`, `office_read`/`office_edit`, `/hooks`, `EnforcedHooks` policy. |
| 2.0.3 | `bobshell` | Background processes, `PreCompact`/`PostCompact` hooks, HTTPS hook handlers, `--disable-mcp` honoured, git fsmonitor off in untrusted folders. |
| 2.0.5 | `bobshell` | `locale` setting, ACP tasks shared with editors, `/manage-secrets set` and `rm`. Recoverable errors no longer end the session. |
| 2.0.6 | `bobshell` | `--max-cost 0` and `--max-turns 0` accepted, prompt allowed after valueless flags, `/export`, Ctrl+Y toggles auto-approve, stricter outside-workspace access in non-interactive runs, ACP `_meta: {"bob/mode": ...}`. The version used for the live local checks. |
| 2.0.7 | migrates to `@ibm/bob` | Installing it migrates the install and updates to 2.0.9 from npm (docs). |
| 2.0.8 | `@ibm/bob` | ACP server reliability fixes. |
| 2.0.9 | `@ibm/bob` | Auto-update for npm installs of `@ibm/bob`. |
| 2.1.0 | `@ibm/bob` | Verified live: licence dialog on a fresh `HOME`, otherwise a run completed (verified 2.1.0). Two dialogs still ask under `--auto-approve --trust`: command security and protected Bob files (observed 2.1.0, see `bob-approvals-and-trust`). |

## Observed live differences
- Package name: a process matcher has to accept both `bobshell` (2.0.6) and `@ibm/bob` (2.0.9, 2.1.0). Match by entry script, not by one name.
- Placeholder colour of the idle composer: truecolor `38;2;118;118;118` or 256-colour `38;5;243` on 2.0.6 and 2.1.0 (default theme); `#a8a8a8` is the dark-theme dim colour in the bundle (not seen live).
- Locale: UI, help text and dialogs follow `LC_ALL`, `LC_MESSAGES`, `LANG` (verified 2.0.6 and 2.1.0 under de_DE and en_US).
- Hooks: no `SessionEnd`, no `StopFailure`; `SessionStart` does not fire at TUI start (verified 2.0.6). `UserPromptSubmit` and `Stop` work (verified 2.0.6, 2.1.0).
- Licence consent: top-level `licenseConsent` in settings (verified 2.1.0).
- Auto-update: on a 2.0.6 host a Bob start installed 2.0.7 into another node prefix. On a 2.1.0 host the legacy migration worker ran `npm install -g --force @ibm/bob@<ver>` then `npm uninstall -g bobshell`, leaving the older binary first on PATH for login shells (verified).

## Local help versus docs (2.0.6 CLI)
- `--log-level`: local `debug, info, warn, error, silent`; docs add `trace`. Use local.
- `--max-cost <number>` locally, `<bobcoins>` in docs; unit is Bobcoins.
- `--config-dir` and `--command-security` appear in the docs for `bob run` but not in the local 2.0.6 `bob run --help`; check `bob run --help` on the target version before relying on them.
- `bob --help` lists a top-level `-p, --prompt`; `run -p` is documented in Bob's README but rejected, the prompt is positional.
- Local help text is German on a de_DE host; flags themselves are unchanged.

## What to do after an upgrade
1. `bob --version` on the exact binary your launcher uses (absolute path).
2. Re-check the version-dependent facts: composer placeholder colour, hook firing, trust handling, the dialogs that remain under auto-approve, `-f json` field names.
3. Capture one `-f stream-json` sample and compare event shapes with the notes in `bob-verify-output`.
