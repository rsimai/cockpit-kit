# Guardrails for kit tool execution

## Background

When kit runs as an ACP server (`kit acp`) it can invoke tools with real side
effects: bash commands, file edits, subagent spawning, etc. The cockpit-kit
frontend needs a reliable mechanism to prevent or gate destructive operations
and — ideally — let the user approve them from the Cockpit UI.

kit's extension API (`~/git/kit/internal/extensions/api.go`) provides the
following relevant primitives:

- `api.OnToolCall(handler)` — fires **before** a tool executes; returning
  `{Block: true, Reason: "..."}` prevents execution entirely.
- `api.OnBeforeAgentStart(handler)` — fires once before the agent loop starts.
- `ctx.GetAllTools()` / `ctx.SetActiveTools(names)` — inspect and restrict the
  active tool set for a session.
- `ctx.PromptConfirm(config)` — shows a yes/no dialog and **blocks** until
  answered. **Currently stubbed to return `{Cancelled: true}` in ACP mode**
  (see `~/git/kit/internal/acpserver/session.go` lines 103–104).

Tool calls carry a `ToolKind` field pre-classified by kit:

| Kind | Examples |
|---|---|
| `execute` | Bash, shell commands |
| `edit` | File write, patch, delete |
| `read` | File read, cat |
| `search` | Grep, glob, find |
| `agent` | Subagent spawning |

---

## Option A — `SetActiveTools` whitelist

**Kit changes required:** none  
**Cockpit UI interaction:** none (policy set at spawn time)

At session start, an extension restricts the active tool set to safe kinds only:

```go
api.OnBeforeAgentStart(func(e ext.BeforeAgentStartEvent, ctx ext.Context) *ext.BeforeAgentStartResult {
    safe := []string{}
    for _, t := range ctx.GetAllTools() {
        if t.Kind == "read" || t.Kind == "search" {
            safe = append(safe, t.Name)
        }
    }
    ctx.SetActiveTools(safe)
    return nil
})
```

Cockpit controls policy by passing an env var or flag when spawning `kit acp`
(e.g. `KIT_ALLOW_KINDS=read,search`), which the extension reads at init time.

**Pros:** Zero kit changes; permanent guarantee for the session; no latency on
tool calls; composable (combine "read+search" for read-only, add "edit" for
supervised edit mode, etc.)

**Cons:** Binary — no per-call user approval. Unsuitable when the user wants to
allow *some* execute calls interactively.

---

## Option B — `OnToolCall` block by kind

**Kit changes required:** none  
**Cockpit UI interaction:** blocked calls surface as failed tool updates

An extension intercepts every tool call and blocks by kind:

```go
api.OnToolCall(func(tc ext.ToolCallEvent, ctx ext.Context) *ext.ToolCallResult {
    if tc.ToolKind == "execute" || tc.ToolKind == "edit" {
        return &ext.ToolCallResult{
            Block:  true,
            Reason: "blocked by cockpit-kit guardrail policy",
        }
    }
    return nil
})
```

The block reason is forwarded to Cockpit as a `toolCall` update with status
`failed`. Cockpit can display this in the UI. Policy can be made dynamic by
reading a kit option (`ctx.GetOption("allow-execute")`) that Cockpit sets
before each prompt via a preparatory extension call.

**Pros:** Zero kit changes; finer-grained than Option A (can allow reads while
blocking writes); blocked calls are visible in the UI with a reason string.

**Cons:** Still no real interactive approval — the user sees the block after the
fact, not a "do you want to allow this?" prompt.

---

## Option C — `PromptConfirm` over ACP  *(recommended long-term)*

**Kit changes required:** ~50 lines in `acpserver/session.go` + `agent.go`  
**Cockpit UI interaction:** full interactive approval per tool call

The `PromptConfirm` stub in the ACP server is replaced with a round-trip
through the ACP protocol to the Cockpit frontend:

1. `OnToolCall` fires on the extension → extension calls `ctx.PromptConfirm()`
   → blocks on a Go channel.
2. ACP server sends a new `session/update` notification of type `confirmRequest`
   to Cockpit, carrying the tool name, kind, and input.
3. Cockpit displays: *"Allow execution of `bash`: `rm -rf /tmp/foo`?"*
4. User approves or denies → Cockpit sends `session/confirm { sessionId, requestId, approved }`.
5. ACP server resolves the channel → `PromptConfirm` returns `{Value: approved}`
   → extension allows or blocks the tool call.

The existing `~/git/kit/examples/extensions/permission-gate.go` already
implements bash guardrails using `ctx.PromptConfirm()` — with Option C it works
as-is in ACP mode, routing through Cockpit.

**Pros:** Proper interactive UX; entire extension ecosystem (`permission-gate`,
etc.) works without modification; most flexible.

**Cons:** Requires a small targeted kit modification (does not touch core
logic); adds a new ACP message type that both sides must implement; tool
execution latency depends on user response time.

---

## Decision criteria

| Criterion | Option A | Option B | Option C |
|---|---|---|---|
| Kit changes | none | none | ~50 lines |
| Interactive approval | no | no | yes |
| Reliable (no bypass) | yes | yes | yes |
| Per-call granularity | no | yes | yes |
| Suitable for unattended use | yes | yes | no |
| Deployment complexity | low | low | medium |

**Suggested approach:** implement Option B immediately as the baseline guardrail
(zero-risk, deployable now), then implement Option C once the ACP client layer
in cockpit-kit is functional and the kit change has been reviewed.
