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

## Next steps

1. Scaffold a `src/kit-client.ts` module that wraps `cockpit.spawn()`:
   - holds one persistent `kit` process for the Cockpit session
   - sends JSON-RPC requests via `.input()`
   - buffers and parses newline-delimited stdout via `.stream()`
   - dispatches responses by `id` using a `Map<id, {resolve, reject}>`
   - handles process crash / restart
2. Build a React UI in `src/app.tsx` that uses the client:
   - show service status (running / not found)
   - provide a chat-style or command input
   - stream partial responses to the UI as they arrive
3. Investigate ACP method names exposed by `kit` (list agents, create run, etc.)
   to wire up the UI properly.

## Open questions

- Does `kit` stdout flush per line or does it buffer? (verify before relying on
  streaming; may need `stdbuf -oL kit` in the spawn call)
- ACP JSON-RPC method names / schema — check `~/git/kit` docs or source.
- Process lifetime strategy: one persistent process vs. spawn-per-request.
