---
name: bob-acp-integration
description: Load when an agent or program should drive Bob Shell as an Agent Client Protocol server (`bob acp`) over stdio instead of scraping a TUI or using one-shot `bob run`: handshake, session, prompt, cancel, permissions, trust.
---

# Drive Bob over ACP (`bob acp`)

Use Bob as a structured, multi-turn backend: JSON-RPC over stdin/stdout, no terminal screen reading. Unmarked = docs; `(verified 2.0.6)` = local `bob acp --help`; `(client)` = taken from pi-bob-acp (not public), an ACP client (code and README) written against ACP v1 and tested with a fake child process, not run live here.

## When to load
- You are building or using a client (Pi extension, editor, orchestrator) that talks to Bob programmatically.
- You need cancel, live permission decisions, or event streaming that `bob run` cannot give.
- You want to avoid pane scraping, composer detection and keystroke delivery.

## When NOT to use
- One bounded task with a final answer: `bob run -f json` is simpler (`bob-run-headless`).
- You must see exactly what a human sees in the TUI, or use hooks that fire only in the TUI: `bob-launch-interactive`.
- Bob older than 2.0.8: ACP server fixes landed in 2.0.8 (changelog); the local 2.0.6 has the command but ACP behaviour was not exercised here.

## Start the server
```
env -u BOB_SESSION BOB_API_KEY=... /abs/path/bob acp [--trust] [--auto-approve] [--disable-mcp] [--disable-subagents] [--log-level <lvl>] [--accept-license]
```
Flags (verified 2.0.6): `--log-level` (also env `BOB_LOG_LEVEL`), `--trust`, `--auto-approve`, `--disable-mcp`, `--disable-subagents`, `--accept-license`. There is no `--workspace`, `--mode` or `--resume`; the workspace is the `cwd` you give per session, and the mode comes through the protocol.
- Spawn it as a child with piped stdin/stdout, working directory = project root, absolute path to `bob`. Run in a terminal it appears to hang: it waits for JSON-RPC on stdin.
- stdout carries protocol only, one JSON object per line (client implementations parse it that way); read logs from stderr (inferred).
- Environment is inherited from the launching process, not from an editor terminal; `PATH`, `BOB_API_KEY` and `LANG` must be right at spawn. Restart the thread after changing them.
- Auth happens during `initialize`: `BOB_API_KEY` in the environment is used with no further step. Without a key or stored token the client is asked to run the SSO browser flow, which an unattended agent cannot do.

## Trust
- Bob checks workspace trust before creating a session; an untrusted workspace makes `session/new` fail (docs; `Workspace ... is not trusted`, client).
- Prefer pre-registering that one folder (entry in `~/.bob/trustedFolders.json`, shape in `bob-launch-interactive`, or run `bob chat --trust --workspace <dir>` once) over starting the server with `--trust`.
- `--trust` trusts EVERY workspace the server opens; `--trust --auto-approve` together removes both the folder gate and per-tool approval. Docs: do not put both in a default configuration. The pi-bob-acp default is `acp --trust` plus client-side auto-allow; treat that as a lab default, not a policy.

## Protocol walk (ACP v1, newline-delimited JSON-RPC 2.0)
Client to Bob, in order (client, ACP spec):
1. `initialize` with `{"protocolVersion":1,"clientCapabilities":{"fs":{"readTextFile":true,"writeTextFile":true},"terminal":true},"clientInfo":{"name":"<client>","version":"<v>"}}`.
2. `session/new` with `{"cwd":"/abs/ws","mcpServers":[]}`; result carries `sessionId`, and models/mode info. Pick the starting mode here: `"_meta":{"bob/mode":"<mode id>"}` (2.0.6 changelog, docs). `session/set_mode` changes it later.
3. `session/prompt` with `{"sessionId":"...","prompt":[{"type":"text","text":"..."}]}`. The request resolves when the turn ends; progress arrives as `session/update` notifications (message chunks, thoughts, `tool_call` updates).
4. `session/cancel` (notification) with `{"sessionId":"..."}` interrupts the turn. This is the ACP equivalent of Escape.
5. Kill the child (SIGTERM) to end; a session reset means a new `session/new`.

Requests Bob sends to the client, which you must answer or the turn stalls (client, ACP spec):
- `session/request_permission`: allow/reject a tool call. The editor owns these dialogs; with `--auto-approve` Bob stops asking.
- `fs/read_text_file`, `fs/write_text_file`, `terminal/*`: only if you advertised the capability. Sandbox writes to the session `cwd`.
A client answers failures with a JSON-RPC error (code `-32000`). Use timeouts on every request; a hung response blocks the child.
Serialise prompts per session; a client queues them.

## Driving it without writing a client
- `acpx` (generic headless ACP CLI) has no built-in `bob` target but takes `--agent <command>`: `acpx --agent "/abs/bob acp" --cwd /abs/ws "prompt"`. Flags like `--approve-reads` or `--max-turns` live on the client side. Not run against Bob here (unverified).
- The pi-bob-acp extension already wraps this for Pi (`bob-acp` provider; tools `bob_status`, `bob_session`, `bob_mode`, `bob_tasks`, `bob_doctor`; env `BOB_ACP_COMMAND`, `BOB_ACP_ARGS`, `BOB_ACP_AUTO_ALLOW`).

## MCP and processes
- Both client and Bob can supply MCP servers; both initialise in one Bob session harness. Give each server one owner. If a tool is missing, check client MCP config and Bob's own (`bob-mcp-setup`).
- Each ACP session creates its own harness: a local MCP server may start once per active thread. Cap parallel sessions when servers are heavy (JVM, container, model).
- `--disable-mcp` isolates startup failures.
- Bob runs commands in its own child process, not your terminal; `cd` or `export` in your shell does not reach it.

## Pitfalls
- Reading the server in a terminal looks like a hang; it is waiting for input.
- Not answering `session/request_permission` stalls the turn with no output. Auto-allow is a security decision; decide per run (`bob-approvals-and-trust`).
- Trust failure shows up at `session/new`, not at `initialize`.
- Without an API key, auth prompts for a browser SSO flow: set `BOB_API_KEY` at spawn, not later.
- Never print the child's environment or the `initialize` traffic if it carries credentials; the client code reviewed never logs `BOB_API_KEY`.
- ACP tasks are shared with the CLI (2.0.5): they show up in `--list-tasks` and can be resumed (`bob-sessions-resume`).
- Protocol field names above come from the ACP spec and a client that has not been run against live Bob in this task; log both directions at debug level on the first run and compare.

## See also
`bob-run-headless`, `bob-launch-interactive`, `bob-approvals-and-trust`, `bob-sessions-resume`, `bob-mcp-setup`, `bob-modes-and-orchestration`.

## Sources
`shell/features/acp`, `shell/changelog` (2.0.2, 2.0.5, 2.0.6, 2.0.8), `local-help` (`bob acp --help`, bob 2.0.6), the README and source of pi-bob-acp.
