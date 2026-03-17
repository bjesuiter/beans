# Beans agent roundtrip example

This note explains the difference between:

- the **old Claude-shaped model** described in `../01-research/claude-code-touchpoints.md`
- the **new minimal Beans agent model** described in `./beans-agent-spec.md`

using one concrete roundtrip:

> user starts work on a bean, collaborates with the agent in the worktree, and later joins that work back into main.

---

## Scenario

Bean:

- `bean-1234`
- title: `Add dark mode toggle`

User goal:

1. start work on the bean
2. ask the agent to implement it
3. answer any agent questions
4. review/finish the work
5. join the work back into main

---

## 1. Old model: Claude-shaped roundtrip

## A. Start work

Beans creates a worktree and opens agent chat.

But the runtime model is already Claude-specific:

- session state includes Claude fields like:
  - `PlanMode`
  - `ActMode`
  - `PendingInteraction`
  - `SessionID`
- the prompt/context tells Claude how to behave inside Beans
- the backend assumes Claude process spawning and Claude resume semantics

So “start work” is not just a Beans workflow step.
It is also implicitly the start of a Claude-shaped session.

## B. User sends first message

Example:

> "Implement the dark mode toggle for settings and wire persistence."

Beans then:

- spawns `claude`
- builds Claude CLI flags
- writes Claude `stream-json` stdin
- persists the user message in Claude-oriented session storage

The UI looks generic, but the execution path is Claude-specific.

## C. Claude works

Beans parses Claude-specific stdout shapes:

- `assistant`
- `content_block_delta`
- `tool_use`
- `result`
- `system.status`
- `task_progress`

UI state is therefore built from Claude-oriented concepts like:

- `systemStatus`
- `subagentActivities`
- tool summary chat messages

## D. Claude asks something

If Claude needs input, it does so through Claude-specific tools.

Examples:

- `AskUserQuestion`
- `ExitPlanMode`
- `EnterPlanMode`

Beans recognizes those exact tool names and applies special logic.

### Ask-user example

Claude asks whether the theme should default to system or dark.

Beans:

- detects `AskUserQuestion`
- parses tool input JSON
- stores `PendingInteraction`
- stops the Claude process
- waits for the user reply

### Plan-exit example

Claude wants to leave plan mode.

Beans:

- detects `ExitPlanMode`
- may read `~/.claude/plans/*.md`
- stores `PendingInteraction`
- stops Claude
- UI shows approval button

Then the frontend does a Claude-shaped sequence:

1. `planMode = false`
2. `actMode = true`
3. send `"yes, proceed"`

That is not a generic agent interaction model; it is a Claude-specific resume flow.

## E. Claude finishes

Claude emits `result`.

Beans:

- persists assistant messages
- persists Claude `session_id`
- sets session idle
- may prune attachments after `/compact`

So the persisted session is still essentially “Claude session state plus Beans wrapping”.

## F. User joins work back into main

At this point the user may trigger commit/review/integrate actions.

In the old model, this can still feel like:

> inject another prompt into Claude and let Claude handle the worktree/git workflow.

That means the workflow risks being shaped around the agent runtime instead of around Beans itself.

---

## 2. New model: minimal Beans-native roundtrip

## A. Start work

Beans creates the worktree exactly as before.

But now the agent side starts from a generic session abstraction:

- `BeansSessionID = bean-1234`
- `WorkingDir = <worktree path>`
- `InitialModeID = "act"`
- driver chosen separately:
  - Claude driver
  - ACP/OpenCode driver
  - pi driver
  - Codex driver

Important difference:

- starting work is a **Beans workflow event**
- opening the agent is a separate driver action via `Open(...)`

## B. User sends first message

Same message:

> "Implement the dark mode toggle for settings and wire persistence."

Beans now calls:

- `LiveSession.Send(...)`

What happens underneath depends on the driver:

- Claude -> Claude stdin JSONL
- ACP -> `session/prompt`
- pi -> `prompt`
- Codex -> `codex` / `codex-reply`

But Beans itself does not care.

## C. Agent works

Instead of the manager parsing Claude-specific structures directly into UI state, the driver emits normalized events like:

- `SessionStatusChanged(running)`
- `MessageStarted`
- `MessageDelta`
- `ToolCallStarted`
- `ToolCallUpdated`
- `StatusTextUpdated`

The reducer turns those into canonical `SessionState`.

So the UI sees Beans-native state like:

- `messages`
- `toolCalls`
- `statusText`
- `currentModeId`
- `pendingRequests`

not Claude-specific backend fields.

## D. Agent needs user input

If the runtime needs user input, the driver emits:

- `InteractionRequested`

Examples:

- kind: `select`
  - options: `system`, `dark`
- kind: `confirm`
  - content: plan text
- kind: `permission`
  - for ACP-style permission flow

Beans then:

1. stores it in `PendingRequests`
2. exposes it through GraphQL
3. UI renders it
4. user answers
5. Beans calls:
   - `Respond(requestID, reply)`

This is the same outside shape whether the source was:

- Claude `AskUserQuestion`
- Claude plan approval
- ACP `session/request_permission`
- pi extension UI request

So the UI is stable even if the runtime changes.

## E. Agent finishes

The driver emits:

- `MessageCompleted`
- `SessionStatusChanged(idle)`
- `ResumeStateUpdated(...)` when needed

Beans persists:

- canonical message history
- canonical tool-call state if used
- opaque adapter-owned resume state

Examples:

- Claude -> session id
- ACP -> ACP session id
- pi -> session file/id
- Codex -> thread id

This means persistence is Beans-owned first, adapter-owned second.

## F. User joins work back into main

This is the biggest conceptual cleanup.

Under the new model, “join work back into main” is **not** part of the agent abstraction itself.

Instead:

- Beans owns the bean/worktree/main-branch lifecycle
- the agent abstraction owns only the runtime/session layer during that lifecycle

So merge-back becomes:

### Beans workflow layer

- mark bean complete
- inspect changes
- integrate worktree into main
- clean up worktree if desired

### Agent session layer

- optionally helps with coding, review, or explanations
- but is not the architectural center of the merge workflow

That is a healthier boundary.

---

## 3. The difference in one sentence

## Old model

The roundtrip is effectively a **Claude-shaped workflow wrapped in generic names**.

## New model

The roundtrip is a **Beans workflow**, and the agent is just a pluggable runtime used during that workflow.

---

## 4. Side-by-side summary

| Step | Old model | New model |
|---|---|---|
| Start work | Creates worktree + implicitly Claude-shaped session | Creates worktree + opens generic Beans session through chosen driver |
| Send prompt | Write Claude `stream-json` stdin | Call `Send(...)` |
| Streaming | Parse Claude events directly | Driver emits normalized events |
| Modes | `planMode` / `actMode` booleans | `currentModeId` + `availableModes` |
| Questions/approval | Claude tool names like `AskUserQuestion`, `ExitPlanMode` | Generic `InteractionRequested` + `Respond(...)` |
| Status/progress | `systemStatus`, `subagentActivities` | `statusText`, `toolCalls` |
| Resume | Claude `session_id` semantics | Opaque `ResumeState` per driver |
| Join back into main | Often agent-flow-shaped | Beans workflow owns it; agent is optional support |

---

## 5. Main takeaway

The new spec separates two concerns that are blurred in the old implementation:

1. **Beans workflow/domain behavior**
   - beans
   - worktrees
   - integration into main

2. **Agent runtime access**
   - prompts
   - messages
   - modes
   - interactions
   - resume state

That separation is the main reason the new abstraction is better.
