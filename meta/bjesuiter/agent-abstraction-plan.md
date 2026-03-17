# Agent abstraction plan for Beans

This is a first proposal for a **protocol-agnostic agent abstraction** in Beans that can cover:

- `meta/docs/beans-claude/current-integration.md` — current Claude Code CLI integration
- `meta/docs/acp/` — Agent Client Protocol
- `meta/docs/pi-rcp/rpc.md` — pi RPC mode

The goal is **not** to force all three protocols into one wire format.
The goal is to define:

1. a **single internal Go interface** that Beans can program against
2. a **single external Beans API contract** that the web UI can use
3. adapter implementations for each concrete protocol/runtime

---

## 1. Summary

My recommendation is:

- make Beans define its own **canonical session model + event model**
- make each runtime/protocol implement a **driver/session adapter**
- move protocol parsing and protocol-specific quirks to adapters
- make the Beans manager/reducer own the UI-facing state
- make the GraphQL API **capability-driven and mode-driven**, not Claude-shaped

In short:

> **Beans should not expose Claude, ACP, or pi-RPC directly.**
> It should expose a Beans-native agent session model that those protocols can map into.

---

## 2. Why the current shape is too specific

Today the implementation leaks Claude concepts into both the backend state and the API:

- `planMode` / `actMode`
- `pendingInteraction` with `EXIT_PLAN` / `ASK_USER`
- `/compact` as a message-level control
- Claude-specific session resume semantics
- Claude-specific tool names
- Claude-specific subagent progress

That works for the current adapter, but it does **not** fit ACP or pi-RPC cleanly.

### ACP mismatch

ACP is centered around:

- explicit `session/new`, `session/load`, `session/prompt`
- `session/update` notifications
- generic `availableModes` / `currentModeId`
- generic `tool_call` lifecycle
- generic `session/request_permission`
- client-provided FS and terminal services

ACP does **not** want Beans' core API to be hardcoded to Claude's `plan/act` booleans.

### pi-RPC mismatch

pi RPC exposes:

- prompt / steer / follow-up queueing
- explicit `compact`, `set_model`, `set_thinking_level`
- event streams for messages and tool execution
- extension UI requests (`select`, `confirm`, `input`, `editor`)
- session switching/forking/session naming

pi RPC has **more runtime controls** than ACP, and some are not part of Claude today.

So the right answer is not “choose one protocol as the API”.
The right answer is to define a stable **Beans-native abstraction** that can represent:

- the common core
- optional capabilities
- protocol-specific extensions behind capabilities

---

## 3. Design principles

## 3.1 Canonical model, not protocol passthrough

Beans should reduce all protocols into a canonical model:

- session lifecycle
- prompt input
- message output
- tool execution
- plans
- modes
- interaction requests
- optional runtime controls

## 3.2 Capabilities first

If a protocol/runtime supports a feature, advertise it.
If it does not, the UI disables the control.

Do **not** encode capability assumptions into field names.

## 3.3 Opaque resume state

Beans should not assume that all runtimes resume with a Claude-style `session_id`.

Instead, store an opaque adapter-owned resume payload, e.g.:

- Claude: CLI `session_id`
- ACP: ACP `sessionId`
- pi-RPC: session file path and/or session ID

## 3.4 Reducer architecture

Adapters should emit normalized events.
Beans should reduce those events into session state for GraphQL/subscriptions.

That gives a clean split:

- **adapter** = transport/protocol/runtime integration
- **manager/reducer** = Beans-owned state model
- **GraphQL** = Beans-owned external API

## 3.5 Separate core vs optional controls

The abstraction should distinguish:

- **core session features** every adapter should try to implement
- **optional controls** such as compaction, model switching, queueing, session fork/switch, slash commands

---

## 4. Proposed internal Go abstraction

## 4.1 High-level structure

I would introduce these layers:

1. `agentcore/`
   - protocol-agnostic types
   - reducer/state machine
   - persistence model
2. `agentdrivers/claude/`
   - current Claude Code CLI adapter
3. `agentdrivers/acp/`
   - ACP adapter
4. `agentdrivers/pi/`
   - pi RPC adapter
5. `agentmanager/`
   - Beans session orchestration and pub/sub

---

## 4.2 Core interfaces

### Driver factory

```go
type Driver interface {
    Kind() string
    Open(ctx context.Context, req OpenSessionRequest, host HostServices) (LiveSession, error)
    Resume(ctx context.Context, req ResumeSessionRequest, host HostServices) (LiveSession, error)
}
```

### Live session

```go
type LiveSession interface {
    Info() SessionRuntimeInfo
    Events() <-chan Event

    Send(ctx context.Context, input UserInput, opts SendOptions) error
    Cancel(ctx context.Context) error
    SetMode(ctx context.Context, modeID string) error
    Reply(ctx context.Context, requestID string, reply InteractionReply) error
    Invoke(ctx context.Context, action ActionInvocation) (ActionResult, error)

    Close(ctx context.Context) error
}
```

### Host services

This is the client/runtime boundary that ACP especially needs.

```go
type HostServices interface {
    RequestPermission(ctx context.Context, req PermissionRequest) (PermissionDecision, error)

    ReadTextFile(ctx context.Context, req ReadTextFileRequest) (ReadTextFileResult, error)
    WriteTextFile(ctx context.Context, req WriteTextFileRequest) error

    CreateTerminal(ctx context.Context, req CreateTerminalRequest) (CreateTerminalResult, error)
    GetTerminalOutput(ctx context.Context, req TerminalOutputRequest) (TerminalOutputResult, error)
    WaitTerminalExit(ctx context.Context, req WaitTerminalExitRequest) (TerminalExitResult, error)
    KillTerminal(ctx context.Context, req KillTerminalRequest) error
    ReleaseTerminal(ctx context.Context, req ReleaseTerminalRequest) error

    // Generic user interaction surface for pi extension UI / Claude AskUser / future custom requests.
    RequestInteraction(ctx context.Context, req InteractionRequest) (InteractionReply, error)
}
```

This is important:

- ACP maps directly to `RequestPermission`, FS, and terminal host methods
- Claude adapter will barely use host services today
- pi adapter can use `RequestInteraction` for extension UI flows

---

## 4.3 Session configuration

```go
type OpenSessionRequest struct {
    BeansSessionID string
    WorkingDir     string
    Provider       ProviderRef
    InitialModeID  string
    MCPServers     []MCPServerConfig
    Context        []ContentBlock
    Meta           map[string]any
}

type ResumeSessionRequest struct {
    BeansSessionID string
    WorkingDir     string
    Provider       ProviderRef
    Resume         ResumeState
    Meta           map[string]any
}
```

Where:

```go
type ResumeState struct {
    DriverKind string
    Opaque     map[string]any
}
```

Examples:

- Claude: `{DriverKind:"claude", Opaque:{"sessionId":"abc"}}`
- ACP: `{DriverKind:"acp", Opaque:{"sessionId":"sess_123"}}`
- pi: `{DriverKind:"pi-rpc", Opaque:{"sessionFile":"/path/...","sessionId":"abc"}}`
`

---

## 4.4 Canonical session state

The manager should keep a canonical snapshot like this:

```go
type SessionState struct {
    BeansSessionID string
    DriverKind     string
    Status         SessionStatus
    WorkDir        string

    Capabilities   SessionCapabilities

    CurrentModeID  string
    AvailableModes []Mode

    Messages       []Message
    ToolCalls      []ToolCall
    Plan           *Plan
    PendingRequests []InteractionRequest
    Commands       []CommandDescriptor

    StatusText     string
    LastError      string

    Runtime        RuntimeState
    Resume         *ResumeState
}
```

Key point:

- `CurrentModeID` + `AvailableModes[]` replaces `planMode` / `actMode`
- `PendingRequests[]` replaces Claude-specific `pendingInteraction`
- `ToolCalls[]` replaces Claude-only tool message hacks as the primary execution model
- `Capabilities` tells the UI what controls exist

---

## 4.5 Canonical event model

Adapters should emit normalized events like:

```go
type Event interface{ isEvent() }
```

Recommended event set:

- `SessionOpened`
- `SessionResumed`
- `SessionStatusChanged`
- `ResumeStateUpdated`
- `ModesUpdated`
- `CurrentModeChanged`
- `CommandsUpdated`
- `StatusTextUpdated`
- `PlanUpdated`
- `MessageStarted`
- `MessageDelta`
- `MessageCompleted`
- `ToolCallStarted`
- `ToolCallUpdated`
- `ToolCallCompleted`
- `InteractionRequested`
- `InteractionResolved`
- `RuntimeUpdated`
- `ErrorEvent`
- `SessionEnded`

That event model is the real protocol boundary.

The reducer turns events into `SessionState`.

---

## 5. Canonical content model

I would strongly recommend reusing **ACP/MCP-style content blocks** internally.

That gives Beans a modern and flexible base type for:

- text
- image
- audio
- resource
- resource_link
- thinking
- tool call references
- terminal references
- diffs

### Proposal

Use ACP/MCP-style content blocks for general display content, then define Beans-native wrappers for places where lifecycle matters.

Examples:

```go
type Message struct {
    ID        string
    Role      MessageRole
    Blocks    []ContentBlock
    Timestamp time.Time
    Meta      map[string]any
}

type ToolCall struct {
    ID        string
    Title     string
    Kind      ToolKind
    Status    ToolCallStatus
    Content   []ToolCallContent
    Locations []Location
    RawInput  map[string]any
    RawOutput map[string]any
    Meta      map[string]any
}
```

Why this is a good fit:

- ACP already uses this structure
- pi-RPC messages/tool results can be converted into it
- Claude stdin image/text content already looks Anthropic/MCP-ish
- it avoids inventing yet another content schema

---

## 6. Capabilities model

The external API must be driven by a capability object instead of hardcoded assumptions.

## 6.1 Core session capabilities

```go
type SessionCapabilities struct {
    Resume            bool
    SetMode           bool
    CancelTurn        bool
    SendImages        bool
    Commands          bool
    Plans             bool
    ToolCalls         bool
    Interaction       bool
    PermissionRequest bool
    FileSystem        bool
    Terminal          bool
}
```

## 6.2 Optional runtime controls

```go
type RuntimeCapabilities struct {
    Steering          bool
    FollowUpQueue     bool
    Compact           bool
    AutoCompact       bool
    ModelSelection    bool
    ThinkingLevel     bool
    SessionFork       bool
    SessionSwitch     bool
    SessionNaming     bool
    ExportHTML        bool
    BashCommand       bool
}
```

These are especially needed for pi-RPC.

---

## 7. Proposed external GraphQL/API shape

I would move from a Claude-shaped session API to a capability-driven one.

## 7.1 Session object

Proposed direction:

```graphql
type AgentSession {
  id: ID!
  driverKind: String!
  status: AgentSessionStatus!
  workDir: String

  capabilities: AgentSessionCapabilities!
  runtimeCapabilities: AgentRuntimeCapabilities!

  currentModeId: String
  availableModes: [AgentMode!]!

  messages: [AgentMessage!]!
  toolCalls: [AgentToolCall!]!
  plan: AgentPlan
  pendingRequests: [AgentInteractionRequest!]!
  commands: [AgentCommand!]!

  statusText: String
  error: String
  runtime: AgentRuntimeState!
}
```

### Replace current Claude-specific fields

Replace:

- `planMode`
- `actMode`
- `pendingInteraction`
- `systemStatus`
- `subagentActivities`

With:

- `currentModeId`
- `availableModes`
- `pendingRequests`
- `statusText`
- `toolCalls`
- `runtime`

This is a much better fit for all three protocols.

---

## 7.2 Mutations

### Core mutations

```graphql
sendAgentInput(sessionId: ID!, input: AgentInput!, delivery: AgentDeliveryMode = IMMEDIATE): Boolean!
cancelAgentTurn(sessionId: ID!): Boolean!
setAgentMode(sessionId: ID!, modeId: String!): Boolean!
respondToAgentRequest(sessionId: ID!, requestId: ID!, response: AgentInteractionResponseInput!): Boolean!
clearAgentSession(sessionId: ID!): Boolean!
```

### Optional control mutations

These should exist only as generic runtime actions, not as protocol-specific hacks:

```graphql
invokeAgentAction(sessionId: ID!, action: String!, args: JSON): AgentActionResult!
```

Examples:

- `action = "compact"`
- `action = "setModel"`
- `action = "setThinkingLevel"`
- `action = "setSessionName"`
- `action = "forkSession"`

The UI can discover whether these are available from `runtimeCapabilities`.

---

## 7.3 Delivery modes

One important difference between protocols is message delivery while streaming.

Proposed generic enum:

```graphql
enum AgentDeliveryMode {
  IMMEDIATE
  INTERRUPT_WHEN_POSSIBLE
  AFTER_TURN
}
```

Mapping:

- Claude current: likely `IMMEDIATE` only, with best-effort current stdin behavior
- ACP: `IMMEDIATE` only unless an adapter provides an extension
- pi-RPC:
  - `IMMEDIATE` -> `prompt`
  - `INTERRUPT_WHEN_POSSIBLE` -> `steer`
  - `AFTER_TURN` -> `follow_up`

This is better than encoding pi-specific queue commands into the core API.

---

## 8. Interaction request model

This is where all three systems can meet.

## 8.1 Canonical interaction request

```go
type InteractionRequest struct {
    ID          string
    Kind        InteractionKind
    Title       string
    Description string
    Options     []InteractionOption
    Content     []ContentBlock
    Meta        map[string]any
}
```

Kinds should include at least:

- `permission`
- `confirm`
- `select`
- `multi_select`
- `text_input`
- `editor`
- `custom`

## 8.2 Mapping

### Claude

- `AskUserQuestion` -> `select` / `multi_select` / `text_input`
- `ExitPlanMode` -> `permission` or `confirm` with plan content attached
- `EnterPlanMode` -> agent-driven `CurrentModeChanged` or optional hidden auto-approve flow

### ACP

- `session/request_permission` -> `permission`
- future custom ACP requests via `_meta` or `_` methods -> `custom`

### pi-RPC

- `extension_ui_request.select` -> `select`
- `confirm` -> `confirm`
- `input` -> `text_input`
- `editor` -> `editor`

This lets the UI build a single interaction surface.

---

## 9. Mode model

The current Beans `planMode`/`actMode` booleans should become a generic mode system.

## 9.1 Canonical mode type

```go
type Mode struct {
    ID          string
    Name        string
    Description string
}
```

And session state keeps:

- `CurrentModeID`
- `AvailableModes`

## 9.2 Mapping

### Claude

Expose modes as:

- `plan`
- `act`

Even if Claude internally uses booleans and special tool flows.

### ACP

Map directly from ACP `availableModes` and `currentModeId`.

### pi-RPC

pi-RPC does not have a first-class ACP-like mode system.
Do **not** invent fake modes unless they are stable.

For pi, leave `availableModes` empty unless a concrete provider-specific mode concept exists.
Use runtime controls for things like model/thinking/queue behavior instead.

This distinction matters. Not every runtime concept should be forced into “modes”.

---

## 10. Commands and actions

ACP has advertised commands.
pi-RPC has `get_commands`.
Claude currently has implicit slash commands like `/compact` and possibly future explicit commands.

I would separate:

- **commands**: user-invoked, prompt-like, shown in composer menus
- **actions**: direct API controls that do not go through the natural language prompt path

### Commands

```go
type CommandDescriptor struct {
    Name        string
    Description string
    InputHint   string
    Source      string
    Meta        map[string]any
}
```

### Actions

```go
type ActionDescriptor struct {
    Name        string
    Description string
    ArgsSchema  map[string]any
}
```

Examples:

- command: `/plan`
- command: `/skill:brave-search`
- action: `compact`
- action: `setModel`
- action: `setThinkingLevel`

Recommendation:

- keep user-facing slash/prompt commands as **commands**
- keep explicit operational controls as **actions**

---

## 11. Persistence changes needed in Beans

Current persistence is too Claude-specific because it mainly stores messages plus a Claude session ID.

I would change persistence to store:

1. canonical message/tool/plan history for UI replay
2. adapter resume state
3. driver kind
4. last known modes/capabilities/runtime settings

## 11.1 Proposed persisted metadata

```json
{
  "type": "meta",
  "driverKind": "claude",
  "resume": {
    "driverKind": "claude",
    "opaque": {"sessionId": "abc123"}
  },
  "currentModeId": "act",
  "sessionName": "feature xyz"
}
```

You can keep JSONL for append-only history, but the meta entry format should become adapter-agnostic.

---

## 12. How each adapter would map into the abstraction

## 12.1 Claude adapter

Wrap the current implementation.

### Input mapping

- `Send(IMMEDIATE)` -> stdin `type:user` message JSONL
- images -> current Anthropic-style content blocks
- `Cancel()` -> current process signal/kill
- `SetMode(plan|act)` -> respawn with appropriate flags
- `Invoke(compact)` -> send `/compact` as a normal user input or implement as adapter-native shortcut

### Output mapping

- `assistant` / `text_delta` -> `Message*` events
- `tool_use` + `input_json_delta` -> `ToolCall*` events
- `system.status` -> `StatusTextUpdated`
- `task_progress` -> either `ToolCallUpdated` or a generic activity event folded into tool/runtime state
- `AskUserQuestion`, `ExitPlanMode`, `EnterPlanMode` -> `InteractionRequested` and/or `CurrentModeChanged`
- `result.session_id` -> `ResumeStateUpdated`

### Important adapter-specific cleanup

Do **not** let Claude tool names leak past the adapter.

---

## 12.2 ACP adapter

Beans becomes an ACP **client** and the external agent is the ACP **agent**.

### Open/resume

- `Open()` -> `initialize` if needed, then `session/new`
- `Resume()` -> `session/load` if capability is supported

### Input mapping

- `Send()` -> `session/prompt`
- `SetMode()` -> `session/set_mode`
- `Cancel()` -> `session/cancel`
- `Reply(permission)` -> return result to in-flight `session/request_permission`

### Output mapping

- `session/update.agent_message_chunk` -> `MessageDelta`
- `session/update.plan` -> `PlanUpdated`
- `session/update.tool_call` / `tool_call_update` -> `ToolCall*`
- `current_mode_update` -> `CurrentModeChanged`
- `available_commands_update` -> `CommandsUpdated`

### Host services mapping

ACP is where `HostServices` really matters:

- `session/request_permission`
- `fs/read_text_file`
- `fs/write_text_file`
- `terminal/*`

This is why the host boundary must exist in the abstraction.

---

## 12.3 pi-RPC adapter

### Open/resume

- spawn `pi --mode rpc`
- `Resume()` uses adapter-owned resume state if session persistence is enabled

### Input mapping

- `Send(IMMEDIATE)` -> `prompt`
- `Send(INTERRUPT_WHEN_POSSIBLE)` -> `steer`
- `Send(AFTER_TURN)` -> `follow_up`
- `Cancel()` -> `abort`
- `Invoke(compact)` -> `compact`
- `Invoke(setModel)` -> `set_model`
- `Invoke(setThinkingLevel)` -> `set_thinking_level`
- `Invoke(setSessionName)` -> `set_session_name`
- `Invoke(forkSession)` -> `fork`

### Output mapping

- `message_start/update/end` -> `Message*`
- `tool_execution_*` -> `ToolCall*`
- `agent_start/end`, `turn_start/end` -> status/runtime events
- `extension_ui_request` -> `InteractionRequested`
- `get_commands` data -> `CommandsUpdated`
- `get_state` data -> `RuntimeUpdated`

### Note on pi-specific controls

pi has extra power. That is fine.
Those features should show up as optional runtime capabilities and actions, not as core requirements for every driver.

---

## 13. What the web UI should become

The frontend should stop assuming:

- there are exactly two modes
- approving means “set plan=false, act=true, then send `yes, proceed`”
- `pendingInteraction` only means `EXIT_PLAN` or `ASK_USER`
- compaction is always a literal `/compact` message

Instead it should become:

1. render `availableModes`
2. render `pendingRequests`
3. render `toolCalls`
4. render `plan`
5. render `commands`
6. enable controls based on `capabilities` and `runtimeCapabilities`

This will let the same UI handle Claude, ACP agents, and pi with fewer special cases.

---

## 14. Recommended migration strategy

## Phase 1 — Introduce the canonical core without changing behavior

- add `agentcore` types and reducer
- wrap current Claude code in a `claude` driver
- keep current GraphQL API, but populate it from the canonical reducer

Goal: prove the architecture without changing the UI much.

## Phase 2 — Replace Claude-shaped state with generic state

- add `currentModeId` / `availableModes`
- add `pendingRequests[]`
- add `toolCalls[]`
- add `capabilities` / `runtimeCapabilities`
- keep old fields temporarily for compatibility

Goal: make the API generic enough for another driver.

## Phase 3 — Add ACP driver

- implement ACP client transport and session management
- implement HostServices for permission / fs / terminal
- map ACP events into canonical reducer

Goal: prove the abstraction handles an editor-agent protocol cleanly.

## Phase 4 — Add pi-RPC driver

- implement `pi --mode rpc` adapter
- wire queue delivery modes, compaction, model/thinking controls, extension UI requests

Goal: prove the abstraction also handles a richer headless RPC agent.

## Phase 5 — Remove legacy Claude-specific fields

- delete `planMode`, `actMode`, `pendingInteraction`, `subagentActivities` once UI is migrated
- keep provider-specific metadata only in `_meta` / `meta`

---

## 15. Concrete recommendations

If you want the shortest path that still scales, I would do these exact things first:

1. **Rename the current `agent.Session` model into a canonical session state model**
   - stop treating it as the Claude runtime state directly
2. **Introduce `DriverKind` + `ResumeState` now**
   - even before ACP/pi are implemented
3. **Replace `planMode`/`actMode` with `CurrentModeID` + `AvailableModes` internally**
   - keep compatibility shims for the current GraphQL API temporarily
4. **Replace `PendingInteraction` with a generic `InteractionRequest`**
   - Claude maps into it today
   - ACP/pi will map naturally later
5. **Add `ToolCall` as a first-class state object**
   - do not treat tool activity as just chat messages
6. **Make `compact`, `setModel`, `setThinkingLevel`, etc. runtime actions**
   - not bespoke GraphQL mutations per provider
7. **Make the reducer own the UI state**
   - adapters should only emit normalized events

---

## 16. Open questions to resolve during implementation

1. **Should Beans persist canonical history, native history, or both?**
   - I would persist canonical history + opaque resume state.
2. **Should GraphQL expose raw provider metadata?**
   - probably yes via optional `meta: JSON`, but keep it non-essential.
3. **Do we need multi-session multiplexing inside one process now?**
   - not in the core API. Hide it inside the driver.
4. **Should commands and actions both exist?**
   - yes. They solve different problems.
5. **Should terminal/bash be unified?**
   - yes at the UI/state level as executable activity, but keep provider-specific actions if needed.

---

## 17. Final recommendation

The right abstraction for Beans is:

> **A Beans-native, event-driven agent session model with capability-based controls and driver adapters for Claude, ACP, and pi-RPC.**

Not:

- a Claude-shaped API with more exceptions
- ACP as the internal truth for everything
- pi-RPC as the internal truth for everything
- direct protocol passthrough to the frontend

If you implement only one thing first, make it this:

> **Normalize all agent runtimes into one reducer-fed event model, and make modes/interactions/tool-calls generic.**

That is the move that will unlock the rest.
