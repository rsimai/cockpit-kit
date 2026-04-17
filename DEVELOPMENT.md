# cockpit-kit Development Notes

## Origin

Forked from [cockpit-project/starter-kit](https://github.com/cockpit-project/starter-kit).
All `starter-kit` / `Starter Kit` references replaced with `cockpit-kit` / `Cockpit Kit`.

Remotes:
- `origin`   — git@github.com:rsimai/cockpit-kit.git  (this repo)
- `upstream` — git@github.com:cockpit-project/starter-kit.git  (upstream template)

## Goal

Build a Cockpit frontend for `~/git/kit` — a locally running AI agent — without
modifying the `kit` application itself.

## Integration approach

`kit` supports the **Agent Client Protocol (ACP)** via **JSON-RPC over stdio**.

Use `cockpit.spawn()` to launch `kit` as a subprocess on the managed host and
communicate with it over its stdin/stdout using newline-delimited JSON-RPC frames.
No HTTP server, no port, no network config required.

```
Browser (React in Cockpit)
  └─ cockpit.spawn(["kit", ...])
       ├─ stdin  →  JSON-RPC requests  (newline-delimited)
       └─ stdout ←  JSON-RPC responses (newline-delimited, matched by id)
```

## ACP protocol details

Investigated `~/git/kit/cmd/acp.go` and `~/git/kit/internal/acpserver/agent.go`.
`kit acp` implements JSON-RPC 2.0 over newline-delimited stdio using
`github.com/coder/acp-go-sdk v0.6.3`.

### Client → kit (requests)

| Method | Key params | Notes |
|---|---|---|
| `initialize` | `protocolVersion` | Handshake. Kit responds with `loadSession: true`, image/embedded-context support. |
| `authenticate` | — | No-op for local stdio; always succeeds. |
| `session/new` | `cwd` (required), `mcpServers` | Creates a Kit session. Returns `sessionId`. |
| `session/load` | `sessionId`, `mcpServers` | Resumes a persisted Kit JSONL session. |
| `session/prompt` | `sessionId`, `prompt[]` | Runs a turn. Prompt is an array of content blocks (`text`, `image`, `resource`). Streams updates; returns `stopReason`: `endTurn` \| `cancelled`. |
| `session/cancel` | `sessionId` | Cancels the in-progress prompt for a session. |
| `session/setModel` | `sessionId`, `modelId` | Swaps the active LLM model mid-session. |
| `session/setMode` | `sessionId`, `mode` | Accepted but currently a no-op in kit. |

### kit → client (streaming notifications)

Sent as `session/update` notifications during a `session/prompt` call:

| Update type | Meaning |
|---|---|
| `agentMessage` text delta | Streamed LLM response text |
| `agentThought` text delta | Streamed reasoning / thinking text |
| `toolCall` start | Tool invocation begun — includes tool name and raw input args |
| `toolCall` update | Tool result — status (`completed` / `failed`) and output content |

### Typical session sequence

```
→ initialize
← InitializeResponse { protocolVersion, agentCapabilities, agentInfo }

→ session/new { cwd: "/path/to/project" }
← { sessionId: "abc-123" }

→ session/prompt { sessionId: "abc-123", prompt: [{ text: { text: "…" } }] }
← session/update (agentMessage delta) × N
← session/update (toolCall start/update) × M   ← zero or more
← PromptResponse { stopReason: "endTurn" }
```

### Notes from the source

- `kit` normalizes incoming `session/new` and `session/load` messages to inject
  `mcpServers: []` if omitted — so we can safely omit it from our client.
- `cwd` is **required** on `session/new`; use the working directory the user
  wants kit to operate in (e.g. their home dir or a project path).
- Tool call IDs may be empty from kit; the server auto-generates `tc_N` ids in
  that case — handle either form on the client side.
- No method exists to list available tools; that is kit's internal concern.

## Next steps

1. Scaffold `src/kit-client.ts` — wraps `cockpit.spawn(["kit", "acp"])`:
   - one persistent process per Cockpit session
   - sends JSON-RPC via `.input(line + "\n", true)`
   - parses newline-delimited stdout via `.stream()`, dispatches by `id`
     using `Map<id, {resolve, reject}>` for requests and a callback for
     `session/update` notifications
   - handles process exit / restart
2. Build React UI in `src/app.tsx`:
   - show kit status (found / not found / running)
   - `session/new` on mount, `session/prompt` on submit
   - stream `agentMessage` deltas into displayed text
   - show tool calls as collapsible items
3. Decide working directory strategy (fixed home dir vs. user-selectable path).

## Open questions

- Does `kit` stdout flush per line? If not, wrap with `stdbuf -oL kit acp` in
  the spawn call.
- Process lifetime: restart on exit, or surface error and let user retry?
- Where should the session's `cwd` come from — fixed, configurable, or picked
  in the UI?
