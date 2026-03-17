# Current Claude integration in Beans

This document describes the **current, implemented** Claude Code integration in Beans as an API/specification reference.

It is based on the current code in:

- `internal/agent/claude.go`
- `internal/agent/manager.go`
- `internal/agent/parse.go`
- `internal/agent/store.go`
- `internal/agent/types.go`
- `internal/agent/describe.go`
- `internal/graph/schema.graphqls`
- `internal/graph/schema.resolvers.go`
- `internal/graph/agent_helpers.go`

It describes what Beans actually does today, not a future generic agent abstraction.

---

## 1. Scope and terminology

Beans exposes the integration through generic API names like `agentSession` and `sendAgentMessage`, but the implementation is currently Claude Code-specific.

There are three layers:

1. **Frontend/GraphQL API**
   - Web UI sends messages and subscribes to session state.
2. **Agent manager/runtime**
   - One in-memory session per bean/worktree ID.
   - Optional persistence to `.beans/.conversations/`.
3. **Claude Code CLI process**
   - Beans spawns `claude` directly.
   - Uses Claude Code `stream-json` for stdin/stdout.

A “bean ID” in this document is also the agent session key. For worktree chats, it is the worktree/bean ID. For the central workspace chat, the special ID is:

```text
__central__
```

Source: `internal/graph/resolver.go`

---

## 2. External API surface used by the web UI

### 2.1 GraphQL query

```graphql
query {
  agentSession(beanId: ID!): AgentSession
}
```

Returns the current materialized session, or `null` if none exists.

### 2.2 GraphQL subscription

```graphql
subscription {
  agentSessionChanged(beanId: ID!): AgentSession!
}
```

Semantics:

- Emits the current session immediately if one exists.
- Emits the full session object on every update.
- If the session is cleared, the subscription emits an empty fallback session:
  - `beanId = requested beanId`
  - `agentType = "claude"`
  - `status = IDLE`
  - `messages = []`

### 2.3 GraphQL mutations

```graphql
mutation {
  sendAgentMessage(beanId: ID!, message: String!, images: [ImageInput!]): Boolean!
  stopAgent(beanId: ID!): Boolean!
  setAgentPlanMode(beanId: ID!, planMode: Boolean!): Boolean!
  setAgentActMode(beanId: ID!, actMode: Boolean!): Boolean!
  clearAgentSession(beanId: ID!): Boolean!
}
```

There is also a testing/helper mutation:

```graphql
mutation {
  setAgentPendingInteraction(beanId: ID!, type: InteractionType!, planContent: String): Boolean!
}
```

### 2.4 AgentSession GraphQL shape

Current frontend-facing shape:

```graphql
type AgentSession {
  beanId: ID!
  agentType: String!
  status: AgentSessionStatus!
  messages: [AgentMessage!]!
  error: String
  planMode: Boolean!
  actMode: Boolean!
  systemStatus: String
  pendingInteraction: PendingInteraction
  workDir: String
  subagentActivities: [SubagentActivity!]!
}
```

Important note: although the names are generic, the fields are Claude-shaped:

- `planMode`
- `actMode`
- `pendingInteraction`
- `systemStatus`
- `subagentActivities`

---

## 3. Claude CLI process contract

## 3.1 Process executable

Beans runs the Claude Code CLI directly:

```text
claude
```

via `exec.CommandContext(ctx, "claude", args...)`.

### 3.2 Working directory

The process working directory is:

- the project root for `__central__`
- the worktree path for worktree sessions

This value is stored on the session as `Session.WorkDir` and assigned to `cmd.Dir`.

### 3.3 Environment

Beans forwards the current process environment with one special rule:

- any environment variable starting with `CLAUDECODE=` is removed

This is done to allow nested Claude sessions.

### 3.4 CLI flags

Beans builds Claude arguments with this base set:

```text
-p
--verbose
--output-format stream-json
--input-format stream-json
--include-partial-messages
--disallowedTools EnterWorktree ExitWorktree
```

Then mode/session flags are appended.

#### Act mode

If `session.ActMode == true`, Beans appends:

```text
--dangerously-skip-permissions
```

#### Plan mode

Else, if `session.PlanMode == true`, Beans appends:

```text
--permission-mode plan
```

#### Resume

If `session.SessionID != ""`, Beans appends:

```text
--resume <sessionID>
```

### 3.5 Effective invocation examples

#### Act mode, fresh session

```bash
claude \
  -p \
  --verbose \
  --output-format stream-json \
  --input-format stream-json \
  --include-partial-messages \
  --disallowedTools EnterWorktree ExitWorktree \
  --dangerously-skip-permissions
```

#### Plan mode, resumed session

```bash
claude \
  -p \
  --verbose \
  --output-format stream-json \
  --input-format stream-json \
  --include-partial-messages \
  --disallowedTools EnterWorktree ExitWorktree \
  --permission-mode plan \
  --resume <session-id>
```

### 3.6 Precedence rule

If both `ActMode` and `PlanMode` were ever set true, **act mode wins** because argument building checks `ActMode` first.

In normal UI usage they are treated as mutually exclusive.

### 3.7 Stdio behavior

- **stdin**: Beans writes Claude `stream-json` input lines.
- **stdout**: Beans reads Claude `stream-json` output line-by-line.
- **stderr**: Beans drains and discards it. Verbose progress written there is not surfaced to the UI.

### 3.8 Process shutdown behavior

Beans uses two termination styles:

- `signal()`
  - closes stdin
  - sends `SIGINT`
  - non-blocking
- `kill()`
  - calls `signal()`
  - waits up to 3 seconds for clean exit
  - then cancels context / force-kills if needed

The graceful stop is important because Claude session state must be saved for `--resume` to work.

---

## 4. Session management model

## 4.1 Session keying

One `agent.Session` is stored per `beanID`.

```go
type Session struct {
  ID                 string
  AgentType          string // always "claude" today
  SessionID          string // Claude CLI resume ID
  Status             SessionStatus
  Messages           []Message
  Error              string
  WorkDir            string
  PlanMode           bool
  ActMode            bool
  SystemStatus       string
  ToolInvocations    []ToolInvocation
  PendingInteraction *PendingInteraction
  SubagentActivities []*SubagentActivity
}
```

### 4.2 In-memory state

The manager keeps:

- `sessions map[string]*Session`
- `processes map[string]*runningProcess`

A session may exist with no running process.

### 4.3 Default mode

New sessions get a default mode from `agent.NewManager(..., defaultMode)`:

- default is `act`
- optional alternative is `plan`

Applied as:

- act default: `PlanMode=false`, `ActMode=true`
- plan default: `PlanMode=true`, `ActMode=false`

### 4.4 Session materialization from disk

If `GetSession(beanID)` is called and no in-memory session exists, Beans attempts to load:

```text
.beans/.conversations/<beanID>.jsonl
```

If messages are found, it creates an in-memory session with:

- `AgentType = "claude"`
- `Status = idle`
- loaded `Messages`
- loaded `SessionID`
- default plan/act mode re-applied in memory

### 4.5 Message send lifecycle

`SendMessage(beanID, workDir, message, images)` does the following:

1. Saves uploaded images to disk if present.
2. Loads or creates the session.
3. Ensures `WorkDir` is set.
4. Appends a new user message.
5. Clears turn-scoped state:
   - `Error = ""`
   - `PendingInteraction = nil`
   - `ToolInvocations = nil`
6. Persists the user message to JSONL.
7. Sets `Status = running`.
8. If a Claude process is already running for this session:
   - writes the new message to existing stdin
9. Otherwise:
   - spawns a new Claude process

Important: Beans supports sending a user message to an already-running Claude process. It relies on Claude Code `stream-json` interleaving semantics for this.

### 4.6 First-message context injection

On the **first spawned turn only** (`SessionID == ""`), Beans may prepend context text from a `ContextProvider`.

The effective first prompt sent to Claude becomes:

```text
<context from provider>

---

<user message>
```

This only happens on initial spawn, not on resumed turns.

### 4.7 Status transitions

Current statuses:

- `idle`
- `running`
- `error`

Typical flow:

1. User sends message -> `running`
2. Claude streams output
3. Claude emits `result` -> `idle`
4. Parse/CLI failure -> `error`

Special case: if Claude begins another turn within the same process after previously going idle, Beans will set the session back to `running` when new actionable stream content arrives.

### 4.8 Stop

`StopSession(beanID)`:

- removes the running process from `processes`
- sets session status to `idle`
- gracefully stops the Claude process

The session and persisted messages remain.

### 4.9 Clear session

`ClearSession(beanID)`:

- deletes persisted conversation file first
- removes in-memory session
- stops any running process
- notifies subscribers

This also prevents immediate re-materialization from an old JSONL file.

### 4.10 Shutdown

`Manager.Shutdown()` kills all running Claude processes concurrently.

---

## 5. Persistence contract

If a beans directory is configured, conversations are persisted under:

```text
.beans/.conversations/
```

## 5.1 Conversation file

Per session:

```text
.beans/.conversations/<beanID>.jsonl
```

### JSONL entry shapes

#### Message entry

```json
{
  "type": "message",
  "role": "user|assistant|tool|info",
  "content": "...",
  "images": [
    {
      "id": "<uuid>.<ext>",
      "media_type": "image/png"
    }
  ],
  "diff": "..."
}
```

#### Meta entry

```json
{
  "type": "meta",
  "session_id": "<claude-session-id>"
}
```

Notes:

- `SessionID` is persisted when a `result` event includes it.
- message persistence is append-only JSONL.
- tool messages are persisted lazily once Beans has enough tool input to derive a readable summary.

## 5.2 Image attachments

Stored under:

```text
.beans/.conversations/attachments/<beanID>/<uuid>.<ext>
```

Allowed MIME types:

- `image/jpeg`
- `image/png`
- `image/gif`
- `image/webp`

Maximum size:

- 5 MB

Served to the frontend as:

```text
/api/attachments/<beanID>/<imageID>
```

## 5.3 Attachment pruning

After a `/compact` user message completes, Beans prunes orphaned attachment files no longer referenced by any persisted message in the session.

---

## 6. Claude stdin JSON protocol used by Beans

Beans writes one JSON object per line to Claude stdin.

### 6.1 User message envelope

Current top-level shape:

```json
{
  "type": "user",
  "message": {
    "role": "user",
    "content": "plain text"
  }
}
```

### 6.2 Text + image content blocks

If images are attached, `message.content` becomes an array of Anthropic-style content blocks:

```json
{
  "type": "user",
  "message": {
    "role": "user",
    "content": [
      {
        "type": "text",
        "text": "Please inspect this screenshot"
      },
      {
        "type": "image",
        "source": {
          "type": "base64",
          "media_type": "image/png",
          "data": "<base64>"
        }
      }
    ]
  }
}
```

### 6.3 Important behaviors

- Beans always emits `type: "user"`.
- Images are loaded from the saved attachment files and inlined as base64.
- The UI does **not** use a special “approve interaction” mutation; approvals and answers are sent as ordinary user messages.

Examples:

- exit plan approval -> `"yes, proceed"`
- ask-user response -> selected option label or custom typed response
- compact -> `"/compact"`

---

## 7. Claude stdout JSON protocol recognized by Beans

Beans reads Claude stdout as newline-delimited JSON and normalizes it into internal events.

## 7.1 Top-level event shapes recognized

Beans currently recognizes these top-level `type` values:

- `stream_event`
- `assistant`
- `content_block_delta` (legacy/direct)
- `content_block_start` (legacy/direct)
- `result`
- `error`
- `system`
- `user`

Anything else is treated as unknown/unhandled.

## 7.2 `stream_event`

`stream_event` wraps Anthropic-style events under `event`.

Example:

```json
{
  "type": "stream_event",
  "session_id": "abc",
  "event": {
    "type": "content_block_delta",
    "index": 0,
    "delta": {
      "type": "text_delta",
      "text": "Hello"
    }
  }
}
```

Recognized nested event types:

- `content_block_delta`
- `content_block_start`
- `content_block_stop` (ignored)
- `message_start` (ignored)
- `message_delta` (ignored)
- `message_stop` (ignored)
- `ping` (ignored)

## 7.3 `assistant`

Full assistant message fallback shape:

```json
{
  "type": "assistant",
  "message": {
    "role": "assistant",
    "content": [
      { "type": "text", "text": "Hi" }
    ]
  },
  "session_id": "abc-123"
}
```

Behavior:

- Beans concatenates all `message.content[*].text` blocks.
- Used as a fallback if streaming deltas did not already build the assistant message.
- `session_id` is copied onto the session if present.

## 7.4 `content_block_start`

Recognized shapes:

### Text block start

```json
{
  "type": "content_block_start",
  "content_block": {
    "type": "text",
    "text": ""
  }
}
```

Semantics:

- starts a new assistant text block
- if the current assistant message already has content, Beans inserts `\n\n`

### Tool use start

```json
{
  "type": "content_block_start",
  "content_block": {
    "type": "tool_use",
    "name": "Read"
  }
}
```

Semantics:

- creates a `role=tool` message in chat
- resets streaming target so later assistant text appears after the tool message
- starts accumulating tool input JSON

## 7.5 `content_block_delta`

Recognized delta variants:

### Text delta

```json
{
  "type": "content_block_delta",
  "delta": {
    "type": "text_delta",
    "text": "Hello"
  }
}
```

Semantics:

- appended to the current streaming assistant message

### Tool input delta

```json
{
  "type": "content_block_delta",
  "delta": {
    "type": "input_json_delta",
    "partial_json": "{\"file_path\":"
  }
}
```

Semantics:

- appended into a per-tool string buffer
- Beans attempts partial parsing to derive a human-readable tool summary
- for some tools it extracts structured information like `file_path`

### Thinking/signature delta

Recognized but ignored:

- `thinking_delta`
- `signature_delta`

These do not currently produce visible UI content.

## 7.6 `result`

Success shape:

```json
{
  "type": "result",
  "subtype": "success",
  "is_error": false,
  "session_id": "def-456",
  "result": "...",
  "total_cost_usd": 0.05
}
```

Error shape:

```json
{
  "type": "result",
  "subtype": "error",
  "is_error": true,
  "session_id": "def-456",
  "result": "something broke"
}
```

Success semantics:

- flushes any pending tool message
- updates session `SessionID`
- persists `SessionID` as JSONL meta
- persists the completed streaming assistant message
- resets `streamingIdx`
- if this is still the active process, sets:
  - `Status = idle`
  - `SystemStatus = ""`
  - `SubagentActivities = nil`
- triggers turn-complete callback
- triggers `/compact` attachment pruning when applicable

Error semantics:

- sets session `Status = error`
- stores error text in `Session.Error`

## 7.7 `error`

Shape:

```json
{
  "type": "error",
  "error": {
    "message": "rate limited"
  }
}
```

Semantics:

- maps directly to session error state

## 7.8 `system`

Recognized subtypes:

### Status

```json
{
  "type": "system",
  "subtype": "status",
  "status": "compacting"
}
```

Semantics:

- stored as `Session.SystemStatus`
- surfaced to the frontend as `systemStatus`

### Task progress

```json
{
  "type": "system",
  "subtype": "task_progress",
  "task_id": "abc123",
  "description": "Reading main.go",
  "last_tool_name": "Read"
}
```

Semantics:

- updates or creates a `SubagentActivity`
- keyed by `task_id`
- used for the UI's subagent activity display

## 7.9 `user`

Beans treats top-level `type: "user"` output events as a boundary signal only.

They are recognized as “tool result / user message events” but do not currently create visible chat messages in the UI.

---

## 8. Tool message summarization and derived data

When Claude starts a tool call, Beans creates a visible tool chat message immediately using the tool name.

Then, as `input_json_delta` arrives, Beans tries to rewrite it into a more readable form:

```text
<tool name>: <summary>
```

### 8.1 Summary extraction fields

Current summary field priority:

1. `description`
2. `file_path`
3. `pattern`
4. `command`
5. `query`
6. `skill`
7. `prompt`

Special case:

- `AskUserQuestion.questions[0].question`

### 8.2 File path normalization

If the summary comes from `file_path` and it is inside the session `workDir`, Beans strips the `workDir` prefix before showing it in the UI.

### 8.3 Diff generation for `Write`

For the `Write` tool, Beans:

1. detects `file_path` from tool input
2. reads the file contents before the write lands
3. later extracts `content` from tool input
4. computes a unified diff
5. stores it on the tool message as `Message.Diff`

This diff is surfaced to the frontend on tool messages.

### 8.4 Plan file discovery

For plan-exit flows, Beans scans recent `ToolInvocations` for a `Write` whose input path matches:

```text
~/.claude/plans/*.md
```

Implementation detail: it actually checks for a path containing `/.claude/plans/` and ending in `.md`.

If found, that file is read and attached to the pending interaction as `PlanContent`.

Fallback if not found:

- last non-empty assistant message content

---

## 9. Blocking interactions and mode-switch semantics

Beans treats some Claude tools specially.

## 9.1 Recognized special tools

- `AskUserQuestion`
- `EnterPlanMode`
- `ExitPlanMode`

## 9.2 `AskUserQuestion`

Flow:

1. Claude starts tool `AskUserQuestion`
2. Beans defers handling until tool input JSON is complete enough
3. Beans parses structured questions from:

```json
{
  "questions": [
    {
      "header": "Approach",
      "question": "Which library should we use?",
      "multiSelect": false,
      "options": [
        { "label": "Option A", "description": "Fast but complex" }
      ]
    }
  ]
}
```

4. Beans sets:

```text
PendingInteraction.Type = ask_user
```

5. Beans signals the Claude process to stop
6. Session remains resumable through preserved `SessionID`

User response path:

- the frontend sends an ordinary user message back through `sendAgentMessage`
- for single-select it sends the option label
- for multi-select it sends selected labels joined by `, `
- user may also type a custom response

## 9.3 `ExitPlanMode`

Current implemented behavior:

1. Claude invokes `ExitPlanMode`
2. Beans creates a pending interaction:

```text
PendingInteraction.Type = exit_plan
```

3. Beans tries to attach `PlanContent`
4. Beans signals the process to stop
5. UI shows the plan and offers approval or freeform refinement

Approval path from UI:

1. frontend sets `planMode=false`
2. frontend sets `actMode=true`
3. frontend sends ordinary user message:

```text
yes, proceed
```

The ordering matters in the frontend so the resumed process starts with the new flags.

## 9.4 `EnterPlanMode`

Current implemented behavior is **auto-approval**.

Flow:

1. Claude invokes `EnterPlanMode`
2. Beans toggles `PlanMode = true`
3. Beans signals the current process to stop
4. Beans immediately respawns by sending:

```text
yes, proceed
```

No pending interaction is shown to the user for this path.

## 9.5 Re-intercept prevention on resumed sessions

Beans will not re-trigger mode-switch pending interactions if session state already reflects that the mode change happened. This avoids loops after `--resume`.

---

## 10. Frontend interaction contract

Although this document focuses on the backend/runtime contract, the current frontend behavior is part of the effective API.

### 10.1 Mode controls

Frontend presents exactly two modes:

- `plan`
- `act`

And drives them via two separate mutations:

- `setAgentPlanMode(beanId, bool)`
- `setAgentActMode(beanId, bool)`

### 10.2 Compact

The UI triggers compaction by sending the literal user message:

```text
/compact
```

There is no separate compact mutation or RPC.

### 10.3 Pending interactions

- `EXIT_PLAN` -> shows approval UI and optional plan markdown
- `ASK_USER` -> shows structured options if available
- `ENTER_PLAN` exists in the API type system, but current runtime behavior auto-approves it before it becomes user-facing in normal operation

---

## 11. Separate Claude integration for workspace descriptions

Beans also uses Claude outside the interactive session runtime.

### 11.1 Purpose

Generate a short 3-8 word description from the first user message in a workspace.

### 11.2 Invocation

```bash
claude --print --model haiku
```

### 11.3 Input

The summarization prompt is written to stdin, not passed as a CLI argument.

Behavior:

- first user message is truncated to 500 chars before prompting
- output is trimmed
- surrounding quotes are removed
- timeout is 30 seconds

This description generation is separate from the main streaming agent process.

---

## 12. Known Claude-specific assumptions baked into the integration

These are not just implementation details; they are contract assumptions the current system depends on.

1. **Claude CLI exists on PATH**
2. **Claude supports `stream-json` stdin/stdout**
3. **Claude uses resumable `session_id` values compatible with `--resume`**
4. **Claude supports these startup flags**
   - `--input-format stream-json`
   - `--output-format stream-json`
   - `--include-partial-messages`
   - `--permission-mode plan`
   - `--dangerously-skip-permissions`
5. **Claude emits Anthropic-style content block events**
6. **Claude may emit `system.status` and `system.task_progress` events**
7. **Claude uses tool names that Beans special-cases**
   - `AskUserQuestion`
   - `EnterPlanMode`
   - `ExitPlanMode`
8. **Claude plan files may be written under `~/.claude/plans/`**
9. **Compaction is triggered by the text command `/compact`**
10. **The frontend approval flow is implemented as plain follow-up user messages, not a separate Claude-native approval RPC**

---

## 13. Notable implementation caveats

These are worth knowing when reading or extending the current integration.

1. **`agentType` is effectively constant**
   - Sessions are materialized with `AgentType = "claude"`.
   - The GraphQL API looks generic, but there is no runtime agent polymorphism yet.

2. **`Message.Diff` is currently generated for `Write`, not generally for all edit-like tools**
   - Some comments/schema wording are broader.
   - The implemented diff capture path in `internal/agent/claude.go` is specific to `Write`.

3. **`eventToolResult` is parsed but not currently used to mutate session state**
   - `parse.go` recognizes top-level `user` events as tool-result boundaries.
   - `readOutput()` does not currently switch on that normalized event.

4. **Loaded sessions re-apply the manager default mode in memory**
   - Persisted conversation history restores messages and `SessionID`.
   - Plan/act mode is not persisted independently in the JSONL store.

5. **The runtime contract is broader than the official Anthropic streaming API docs**
   - Beans depends not just on Anthropic-style content blocks.
   - It also depends on Claude Code CLI flags, session resume behavior, and extra top-level events like `system.status` and `system.task_progress`.

---

## 14. Practical summary

Today, Beans' Claude integration is best understood as this contract:

- GraphQL exposes a generic-looking `AgentSession` API.
- The backend manager maintains one session per bean/worktree.
- Each session can spawn or resume a `claude` CLI process.
- Messages are sent through Claude Code `stream-json` stdin.
- Output is parsed from Claude Code `stream-json` stdout.
- Special Claude tools drive plan mode, ask-user prompts, and resumable approval flows.
- Session history and resume IDs are persisted in JSONL.
- The UI is generic in naming, but structurally depends on Claude concepts.

That is the current built implementation.
