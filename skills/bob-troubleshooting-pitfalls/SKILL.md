---
name: bob-troubleshooting-pitfalls
description: Use when a Bob Shell launch, run or worker misbehaves (hangs, nested-session error, license or trust dialog, blocked reads, auto-update side effects, translated UI, 402/403, wrong version) or before launching Bob from inside another Bob session.
---

# Bob Shell troubleshooting and pitfalls

Symptom, cause, fix for the traps a steering agent actually hits when it launches and drives Bob. Facts are marked `docs`, or `(verified <version>)` when seen live or with the local CLI.

## When to load
- Bob does not start, hangs, prints nothing, or the pane never shows a usable composer.
- A tool call is blocked, a hook never fires, or auto-approve seems ignored.
- Behaviour differs between machines or versions (2.0.6, 2.0.9, 2.1.0, old `bobshell`).
- You are about to run `bob` from a process that is itself under Bob.

## First checks (30 seconds)
```bash
command -v bob; bob --version                       # 2.x expected; 1.0.x is a stale install
env | grep -E '^(BOB_SESSION|BOB_API_KEY|BOBSHELL_API_KEY)=' | sed 's/=.*/=<set>/'   # names only, never print values
git -C <ws> rev-parse --show-toplevel               # workspace root Bob will resolve
ls -t ~/.bob/logs/shell/ | head -3                  # newest log; rerun with --log-level debug if empty
```
Debug: `bob run --log-level debug ...` or `BOB_LOG_LEVEL=debug` (docs). Local help lists `debug, info, warn, error, silent`; docs also name `trace`. Prefer local help. Logs rotate: 10 files, 5 MB each (docs).

## Launch and session
- Nested session throws. Symptom: Bob exits at start when launched from a tool call of another Bob. Cause: Bob sets `BOB_SESSION=1` for its tool subprocesses and its source throws on a nested session (verified 2.0.6). Fix: launch with `env -u BOB_SESSION bob ...`. Also clear other agents' markers you do not want inherited (`CLAUDECODE`, `CURSOR_AGENT`, `GEMINI_CLI`).
- `bob run` hangs with no output. Cause: stdin left open (verified 2.0.6, 150 s). Fix: `</dev/null` when the prompt is an argument. `bob --list-tasks` hung about 3 minutes in a real home (killed; likely same cause, not retested): add `</dev/null` and a timeout.
- Two different keys. Symptom: Bob refuses to start. Cause: `BOB_API_KEY` and the older `BOBSHELL_API_KEY` both set to different values (observed with synthetic keys). Fix: export only one. `bob run` needs an API key; SSO tokens from `bob chat` are not reused (docs).
- Wrong or missing team: a `general`-type key needs `--team-id <id>`; Bob never picks one (docs).
- Licence dialog blocks the TUI. Symptom: pane shows the IBM licence text, no composer, a readiness check fails. Cause: fresh `HOME` or new version (verified 2.1.0). Fix: set top-level `"licenseConsent": true` in settings (not nested under `ibm`), or pass `--accept-license`. Read the pane before typing anything.
- Workspace resolves to the wrong folder. Cause: Bob walks up from the cwd to the first `.git`, then `.bob` (`.git` wins since 2.0.1); with neither it uses the cwd (docs). Fix: always pass `--workspace <abs path>` and run from the repo root.
- `bob chat` has no positional prompt (verified 2.0.6): start the TUI, wait for the composer, then type the brief. Use `bob run` for one-shots. Local `bob chat --help` has no `--format`.
- Prompt after flags: before 2.0.6 `bob run --disable-mcp "text"` was rejected; 2.0.6 fixed it (docs). On older builds put the prompt first.

## Trust, approvals, file access
- `--auto-approve` does nothing in an untrusted folder; Bob prompts before every tool and the worker waits forever. Hooks in `.bob/settings.json` are skipped too, so no turn signals (docs, verified 2.0.6). Fix: pass `--trust`, or pre-register the exact path as `TRUST_FOLDER` in `~/.bob/trustedFolders.json`. Interactive `--trust` persists; in `bob run` it applies to that run only (docs).
- `DONT_TRUST` entry makes `bob run` throw "<folder> is not a trusted folder" (docs).
- Reads outside the workspace are blocked by default. Symptom: Bob cannot read the brief you saved in `/tmp` or `~/notes`. Fix: copy the brief into the workspace (write it to `.bob/launch-brief.md` and keep it out of git via `.git/info/exclude`), or enable `approval.permissionOptions[{groupId:"read",enableOutsideWorkspace:true}]` (docs). In non-interactive runs the global setting and the permission group must both allow it (docs 2.0.6).
- Writes to `.bob/settings.json` and `~/.bob/*/settings.json` always need explicit approval, even with auto-approve (docs 2.0.1). Do not plan on Bob editing its own config unattended.
- MCP tools are not auto-approved unless the `mcp` group is granted too (docs 2.0.1).
- `.bobignore` or rules edits seem ignored: allow about 30 s for ignore patterns; `/clear` for new rules (docs).
- Hooks you wrote ran against an untrusted repo: global hooks run in every folder that is not `DONT_TRUST` (docs). Review hooks before pointing Bob at foreign code.

## Auto-update and installs
- Self-update runs `npm install -g --force` against whichever node is active and can write outside the install you meant to use (verified 2.0.6: a second copy, 2.0.7, appeared under another nvm node). Setting is `bobShell.autoUpdate`, read from user settings only (`~/.bob/settings/settings.json`), so you cannot pin it per task. Fix: set it `false` there before automating Bob; a throwaway `HOME` with that setting isolates tests.
- Package names changed: `bobshell` up to 2.0.7, `@ibm/bob` from 2.0.8 (docs). Installing 2.0.7 migrates to `@ibm/bob` and removes `bobshell` (docs; migration worker observed on 2.1.0 host).
- Stale binary on PATH. Symptom: `Unknown arguments: auto-approve, workspace`; cause: a leftover `bobshell` 1.0.x ahead of the new install on a login PATH (verified 2.1.0 host). Fix: `bash -lic 'command -v bob; bob --version'` and launch by absolute path, `node <.../dist/bob.js> chat ...`, because a pane daemon's PATH can differ from yours.
- Uninstall both names if two installs coexist: `npm uninstall -g @ibm/bob` and `npm uninstall -g bobshell`. `rm -rf ~/.bob` deletes settings, trust store and tasks (docs).

## UI and state signals
- UI is translated from `LC_ALL`, `LC_MESSAGES`, then `LANG`, plus the `locale` setting (verified 2.0.6/2.1.0, docs 2.0.5). Help output is German on a de_DE host. Never match English strings in `pretty` output or on screen; pin `LC_ALL=en_US.UTF-8` if you must read text, and prefer `-f json`.
- Placeholder colour differs by version and terminal theme: truecolor `38;2;118;118;118` on 2.0.6 and 2.1.0, `38;5;243` without COLORTERM; a dark-theme variant is 168/248 (seen in the bundle, not live). Classifiers keyed to one colour break after upgrades.
- Stop hook does not fire on a single Escape interrupt (verified 2.0.6). Bob has no `StopFailure` or `SessionEnd` hook and `SessionStart` does not fire at TUI start (verified 2.0.6), so busy state stays "working" after auth errors or crashes. Confirm with the pane, not the hook alone.
- Exit with `/exit`; interrupt a running tool with one Escape (verified 2.0.6). `Ctrl+C` exit hang was fixed in 2.0.1.
- Bob can survive a closed window until it gets SIGTERM; it exits on the first SIGTERM (verified 2.0.6). Interactive login shells ignore SIGTERM, which can look like Bob refusing to die.

## Limits, quota, network
- JSON `401` "Invalid or expired API Key" is a real auth verdict (observed). A `403` with an HTML body from the edge is not; treat it as a transport or proxy problem, not a bad key (observed on a non-Bob client). Budget stop on inference is `402` with separate texts for team budget, suspended plan, expired trial (strings in Bob's translations). Docs: allocation resets each billing cycle; overage is opt-in. Details: `bob-cost-and-limits`.
- `--max-cost 0` and `--max-turns 0` mean unlimited since 2.0.6 and were rejected before (docs). Docs call the unit Bobcoins; the README calls it USD, which is wrong.
- Certificate errors behind a TLS-inspecting proxy ("Unable to verify certificate", `ECONNREFUSED`, `ETIMEDOUT`): set `NODE_EXTRA_CA_CERTS=/abs/path/ca.crt` and `HTTP_PROXY`/`HTTPS_PROXY`/`NO_PROXY` (docs). Avoid `NODE_TLS_REJECT_UNAUTHORIZED=0` except as a temporary test.
- Slow or heavy runs: too much context (large dirs, binaries, `node_modules/`). Add `.bobignore`, use `@file` references, restart long sessions (docs).

## Pitfalls
- Reading "idle" as "done": composer empty plus no hook means nothing without a diff. See `bob-verify-output`.
- Typing into a pane that shows a dialog: the keystrokes answer the dialog (accepting a licence or trust prompt by accident).
- Hard-coding a Bob version or package name in launchers: it moves from `bobshell` to `@ibm/bob` and updates itself.
- Letting auto-update run on a shared machine: it changes global npm state for other tools.
- Assuming project config applies in an untrusted folder: project MCP servers, modes, skills, rules and `AGENTS.md` are not loaded there (docs).

## See also
`bob-verify-output`, `bob-launch-interactive`, `bob-run-headless`, `bob-approvals-and-trust`, `bob-hooks`, `bob-cost-and-limits`, `bob-sessions-resume`, `bob-mcp-setup`.

Version-by-version differences and the install/migration timeline: `references/versions.md`.

## Sources
`shell/troubleshooting/troubleshoot`, `shell/troubleshooting/ts-proxy`, `shell/faq` (questions only, no answers fetched), `shell/changelog`, `shell/getting-started/uninstalling-bobshell`, `shell/core-concepts/tools`, `shell/security/trusted-folders`, `shell/configuration/approval-settings`, `shell/configuration/lifecycle-hooks`, `shell/getting-started/start-bobshell-non-interactive`; local-help (`bob --help`, `bob run --help`, `bob chat --help`, 2.0.6). `shell/configuration/telemetry-data-shell` was read and is not relevant to steering.
