# Agent abstraction plan for Beans

This is the prose plan behind the minimal Beans agent spec.

It covers how Beans should evolve from the current Claude-specific integration toward a small Beans-native abstraction that can support:

- `meta/docs/beans-claude/current-integration.md` — current Claude Code CLI integration
- `meta/docs/acp/` — ACP / OpenCode
- `meta/docs/pi-rcp/rpc.md` — pi RPC mode
- `meta/docs/codex-mcp/mcp-server-exploration.md` — Codex MCP server

The goal is **not** to force all of these protocols into one wire format.
The goal is to define:

1. one small internal Go abstraction
2. one UI-facing Beans session model
3. one reducer-driven state flow
4. concrete drivers per protocol/runtime

---

## 1. Summary

The new direction is intentionally smaller than the earlier drafts.

My recommendation is:

- make Beans define a **minimal canonical session model**
- make each runtime implement a **driver adapter**
- keep **interactions as first-class session events/state**
- keep **host execution requirements optional and backend-only**
- only model in the core what Beans clearly needs today
- add richer concepts later only when a real driver/UI feature needs them

In short:

> **Beans should not expose Claude, ACP, pi-RPC, or Codex MCP directly.**
> It should expose a small Beans-native session model that those runtimes can map into.

---

## 2. Why the current shape is too specific

Today the implementation leaks Claude concepts into both the backend state and the API:

- `planMode` / `actMode`
- `pendingInteraction` with `EXIT_PLAN` / `ASK_USER`
- `/compact` as a message-level control
- Claude-specific resume semantics
- Claude-specific tool names
- Claude-specific subagent progress

That works for the current adapter, but it is too specific to be the long-term model.

What Beans really needs at its center is much smaller:

- open/resume a session
- send input
- cancel/stop
- set mode if supported
- receive streamed output/events
- surface pending interactions
- persist canonical history + resume state

Everything beyond that should be optional.

---

## 3. Design principles

## 3.1 Canonical model, not protocol passthrough

Beans should reduce all runtimes into one canonical session model.

Not every protocol feature belongs in the initial core.
The core should focus on:

- session lifecycle
- messages
- modes
- interactions
- status/error
- resume state
- tool calls

## 3.2 Reducer architecture

Drivers emit normalized events.
Beans reduces those events into session state.
GraphQL and the UI consume that state.

That gives a clean split:

- **driver** = protocol/runtime integration
- **manager/reducer** = Beans-owned state
- **GraphQL** = external Beans API
- **UI** = renderer of Beans session state

## 3.3 Opaque resume state

Beans should not assume every runtime resumes like Claude.

Resume state should stay adapter-owned and opaque, for example:

- Claude: CLI `session_id`
- ACP: ACP `sessionId`
- pi RPC: session file / session ID
- Codex MCP: `threadId`

## 3.4 Interactions are state, not callbacks

This is one of the key decisions.

Beans should not center the abstraction around synchronous host UI callbacks.
Instead:

1. driver emits `InteractionRequested`
2. reducer stores it on the session
3. UI renders it
4. user answers it
5. Beans calls `Respond(...)`

This cleanly fits:

- Claude `AskUserQuestion`
- Claude plan approval
- pi extension UI requests
- ACP permission requests

## 3.5 Host execution requirements are optional

ACP/OpenCode is important, but ACP should **not** become the core model.

Instead, ACP-specific backend needs like:

- file system access
- terminal access
- permission mediation

should live behind a separate backend-only declaration:

- `HostCapabilityRequirements`

That means:

- Claude/pi/Codex do not force ACP host semantics into the core
- ACP can still be added later as a driver with extra backend requirements

## 3.6 Default to `act`

Beans should default the mode to `act`.

If a runtime has no native mode concept, Beans should still expose:

- `currentModeId = "act"`
- `availableModes = [act]`
- `SetMode = false`

That keeps the external model stable without inventing fake mode switching.

---

## 4. Proposed internal Go abstraction

## 4.1 High-level structure

I would introduce these layers:

1. `agentcore/`
   - protocol-agnostic types
   - reducer/state machine
   - persistence model
2. `agentdrivers/claude/`
   - Claude Code CLI adapter
3. `agentdrivers/acp/`
   - ACP/OpenCode adapter
4. `agentdrivers/pi/`
   - pi RPC adapter
5. `agentdrivers/codex/`
   - Codex MCP adapter
6. `agentmanager/`
   - Beans session orchestration and pub/sub

## 4.2 Core interfaces

The base interface should stay small:

```go
type Driver interface {
    Kind() string
    HostCapabilityRequirements() HostCapabilityRequirements
    Open(ctx context.Context, req OpenSessionRequest) (LiveSession, error)
    Resume(ctx context.Context, req ResumeSessionRequest) (LiveSession, error)
}

type LiveSession interface {
    Events() <-chan Event

    Send(ctx context.Context, input UserInput, opts SendOptions) error
    Cancel(ctx context.Context) error
    SetMode(ctx context.Context, modeID string) error
    Respond(ctx context.Context, requestID string, reply InteractionReply) error

    Close(ctx context.Context) error
}
```

Notably removed from the core:

- `Info()`
- direct runtime action invocation
- protocol-specific host service methods

Those can be added later if real usage demands them.

## 4.3 Session configuration

The open/resume requests should also stay small:

```go
type OpenSessionRequest struct {
    BeansSessionID string
    WorkingDir     string
    InitialModeID  string // default: "act"
    Context        []ContentBlock
    Meta           map[string]any
}

type ResumeSessionRequest struct {
    BeansSessionID string
    WorkingDir     string
    Resume         ResumeState
    Meta           map[string]any
}

type ResumeState struct {
    DriverKind string
    Opaque     map[string]any
}
```

This means the earlier ideas like `ProviderRef` or `MCPServers` should **not** be part of the initial core request shape.
If ACP later needs extra setup, it can come through `Meta` or a later extension.

## 4.4 Canonical session state

The reducer-owned session state should also be minimal:

```go
type SessionState struct {
    BeansSessionID  string
    DriverKind      string
    Status          SessionStatus
    WorkDir         string

    Capabilities    SessionCapabilities
    CurrentModeID   string
    AvailableModes  []Mode

    Messages        []Message
    ToolCalls       []ToolCall
    PendingRequests []InteractionRequest

    StatusText      string
    LastError       string
    Resume          *ResumeState
}
```

Notes:

- `CurrentModeID` replaces `planMode` / `actMode`
- `PendingRequests` replaces Claude-specific pending interaction state
- `ToolCalls` remain in the minimal spec because Beans already exposes tool-like activity, diffs, and progress today

What is intentionally not first-class in the minimal spec yet:

- plans
- commands
- runtime actions
- richer runtime capability objects

## 4.5 Minimal event model

Drivers should emit only the events the current Beans model clearly needs:

- `ResumeStateUpdated`
- `SessionStatusChanged`
- `ModesUpdated`
- `StatusTextUpdated`
- `MessageStarted`
- `MessageDelta`
- `MessageCompleted`
- `ToolCallStarted`
- `ToolCallUpdated`
- `ToolCallCompleted`
- `InteractionRequested`
- `InteractionResolved`
- `ErrorEvent`
- `SessionEnded`

Everything else should be added only when a real driver needs it.

---

## 5. Canonical content model

I still think Beans should reuse ACP/MCP-style content blocks internally where practical.

Why:

- ACP already uses them
- Claude image/text input already looks similar
- pi and Codex output can be mapped into them
- it avoids inventing yet another content structure

But the core should not over-design this part yet.
The important thing is that messages and tool calls can carry flexible content blocks when needed.

---

## 6. Capabilities model

The capability model should be simplified as well.

## 6.1 UI-facing session capabilities

```go
type SessionCapabilities struct {
    Resume      bool
    SetMode     bool
    CancelTurn  bool
    SendImages  bool
    ToolCalls   bool
    Interaction bool
}
```

This is enough for the minimal API.

## 6.2 Backend-only host requirements

```go
type HostCapabilityRequirements struct {
    FileSystem  bool
    Terminal    bool
    Permissions bool
}
```

This is **not** part of the core UI-facing API.
It is backend plumbing for ACP-like drivers.

That is the key compromise:

- ACP support stays possible
- ACP does not define the whole abstraction

---

## 7. External GraphQL/API direction

The GraphQL API should move away from the Claude-shaped session model, but the first generic version should stay smaller than earlier drafts.

At minimum, the external session shape should move toward:

- session id / driver kind / status / workdir
- capabilities
- current mode + available modes
- messages
- tool calls
- pending requests
- status text
- error

The current Claude-specific fields to phase out are still:

- `planMode`
- `actMode`
- `pendingInteraction`
- `systemStatus`
- `subagentActivities`

The minimal generic mutations should be:

- send input
- cancel turn
- set mode
- respond to interaction
- clear session

Direct runtime actions like compaction or model switching should be deferred until a real driver/UI feature forces them.

---

## 8. Interaction request model

The interaction model still matters a lot, even in the reduced spec.

At minimum, interaction kinds should cover:

- `permission`
- `confirm`
- `select`
- `multi_select`
- `text_input`
- `editor`
- `custom`

Mapping examples:

### Claude
- `AskUserQuestion` -> `select` / `multi_select` / `text_input`
- exit-plan approval -> `confirm` or `permission`

### ACP
- `session/request_permission` -> `permission`

### pi-RPC
- extension UI `select` -> `select`
- `confirm` -> `confirm`
- `input` -> `text_input`
- `editor` -> `editor`

This is still the cleanest shared surface across runtimes.

---

## 9. Mode model

The mode model stays, but also stays simple.

```go
type Mode struct {
    ID          string
    Name        string
    Description string
}
```

Mapping:

### Claude
- `plan`
- `act`

### ACP
- whatever the agent exposes natively

### pi-RPC
- always expose `act`
- no mode switching

### Codex MCP
- always expose `act`
- no mode switching

---

## 10. What we are explicitly deferring

Compared to earlier drafts, the following are no longer part of the minimal core plan:

- first-class plans
- commands
- direct runtime actions in the base interface
- richer runtime capability objects
- ACP-specific open/session config beyond `Meta`

These may still be added later, but only when a real driver or UI requirement justifies them.

That is a deliberate simplification.

---

## 11. Persistence changes needed in Beans

Current persistence is too Claude-specific because it mainly stores messages plus a Claude session ID.

The new persistence direction should be:

1. canonical message history
2. canonical tool-call history if/when used
3. adapter-owned resume state
4. driver kind
5. enough metadata to rebuild session state safely

The main rule remains:

- persist Beans-owned canonical history
- persist adapter-owned opaque resume state

This is especially important for Codex MCP, where native resume state is not durable across process restarts.

---

## 12. How each adapter would map into the reduced abstraction

## 12.1 Claude adapter

Claude is still the first implementation and the reference migration path.

### Input mapping

- `Send(...)` -> stdin `stream-json` user message
- `Cancel()` -> stop the current process
- `SetMode(plan|act)` -> respawn with appropriate flags
- `Respond(...)` -> usually plain follow-up user input such as `yes, proceed` or selected answers

### Output mapping

- assistant text / deltas -> `Message*`
- tool use -> `ToolCall*`
- system status -> `StatusTextUpdated`
- blocking tool flows -> `InteractionRequested`
- resume session id -> `ResumeStateUpdated`

## 12.2 ACP adapter

ACP should be supported, especially for OpenCode, but as an adapter with extra backend requirements.

### Open/resume

- `Open()` -> `initialize` + `session/new`
- `Resume()` -> `session/load` if supported

### Input mapping

- `Send()` -> `session/prompt`
- `Cancel()` -> `session/cancel`
- `SetMode()` -> `session/set_mode`
- `Respond(...)` -> answer pending permission requests

### Output mapping

- message chunks -> `MessageDelta`
- tool calls -> `ToolCall*`
- mode updates -> `ModesUpdated` / current mode updates
- permission requests -> `InteractionRequested`

### Host requirements

ACP/OpenCode likely declares:

```go
HostCapabilityRequirements{
    FileSystem:  true,
    Terminal:    true,
    Permissions: true,
}
```

This is backend plumbing, not the core model.

## 12.3 pi-RPC adapter

### Open/resume

- spawn `pi --mode rpc`
- adapter-owned resume state if persistence is enabled

### Input mapping

- `Send(...)` -> `prompt`
- queueing behavior like `steer` / `follow_up` can be added later as an optional extension
- `Cancel()` -> `abort`
- `Respond(...)` -> answer extension UI requests when needed

### Output mapping

- message streaming -> `Message*`
- tool execution -> `ToolCall*`
- extension UI requests -> `InteractionRequested`
- always expose mode `act`

## 12.4 Codex MCP adapter

### Open/resume

- spawn one long-lived `codex mcp-server`
- initialize it
- `Open()` -> `tools/call(name="codex")`
- `Resume()` within the same live process -> `tools/call(name="codex-reply")`

### Input mapping

- `Send(...)` -> `codex` for first turn, `codex-reply` for later turns
- `Cancel()` -> unsupported until proven otherwise
- `SetMode()` -> unsupported
- always expose mode `act`

### Output mapping

- final `structuredContent.content` -> `MessageCompleted`
- `threadId` -> `ResumeStateUpdated`
- `codex/event` deltas -> `MessageDelta`
- useful progress events -> status/runtime updates as needed

### Important constraint

`threadId` is process-local.
So canonical Beans persistence matters more than native resume durability.

---

## 13. What the web UI should become

The frontend should stop assuming:

- there are exactly two modes in every runtime
- pending interaction only means Claude plan exit or ask-user
- compaction is always a special command
- every runtime exposes the same control surface

Instead it should:

1. render current mode and available modes
2. render pending requests
3. render messages
4. render tool calls
5. enable controls based on session capabilities

That is enough for the minimal generic API.

---

## 14. Recommended migration strategy

## Phase 1 — Introduce the minimal canonical core

- add `agentcore` types and reducer
- wrap current Claude code in a `claude` driver
- keep current GraphQL API, but populate it from the new reducer

Goal: prove the minimal architecture without big UI churn.

## Phase 2 — Replace Claude-shaped session state

- add `currentModeId` / `availableModes`
- add `pendingRequests[]`
- add `toolCalls[]`
- keep compatibility shims for old GraphQL fields temporarily

Goal: make the API generic enough for a second driver.

## Phase 3 — Add ACP/OpenCode driver

- implement ACP transport/session management
- implement ACP backend plumbing behind `HostCapabilityRequirements`
- map ACP permission requests into interaction state

Goal: support OpenCode without making ACP the core model.

## Phase 4 — Add pi-RPC driver

- implement `pi --mode rpc` adapter
- map message/tool/interaction flows into the reducer
- add richer delivery/runtime features only if the UI actually needs them

## Phase 5 — Add Codex MCP driver

- implement `codex mcp-server` adapter
- handle process-local thread resume state
- rely on canonical Beans persistence for long-term continuity

## Phase 6 — Remove legacy Claude-shaped fields

- delete `planMode`, `actMode`, `pendingInteraction`, `subagentActivities` once UI is migrated

---

## 15. Concrete implementation recommendations

If I were starting implementation now, I would do these first:

1. **Rename the current `agent.Session` model into a canonical reducer-owned session state**
2. **Introduce `DriverKind` + `ResumeState` now**
3. **Replace `planMode`/`actMode` with `CurrentModeID` + `AvailableModes` internally**
4. **Replace `PendingInteraction` with generic `InteractionRequest` state**
5. **Add `ToolCall` as a first-class state object**
6. **Keep the base driver interface small**
7. **Do not add plans/commands/runtime actions until really needed**

---

## 16. Open questions

A few questions remain, but the reduced plan narrows them a lot:

1. Should tool calls be in v1, or added immediately after the minimal migration?
2. How much raw provider metadata should GraphQL expose, if any?
3. How should canonical history be serialized once tool calls become first-class?
4. When pi/ACP land, which richer control surfaces actually need to be promoted into the main API?

---

## 17. Final recommendation

The right abstraction for Beans is:

> **A small Beans-native, event-driven session model with pluggable drivers, first-class interaction state, and optional backend host requirements for ACP-like runtimes.**

Not:

- a Claude-shaped API with more exceptions
- ACP as the internal truth for everything
- pi-RPC as the internal truth for everything
- direct protocol passthrough to the frontend

If you implement only one thing first, make it this:

> **Introduce the reducer-owned minimal session model and migrate Claude into it first.**

That is the step that unlocks everything else.
