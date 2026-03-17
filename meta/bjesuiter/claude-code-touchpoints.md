# Claude Code touchpoints in Beans web UI

This note captures the initial codebase analysis of where and how Claude Code is used today in Beans, before introducing a more generic agent abstraction.

## Executive summary

The web UI is exposed through mostly generic names like `agentSession`, `sendAgentMessage`, and `AgentChat`, but the implementation is still strongly Claude-specific in three layers:

1. **Process/runtime integration**
   - Beans spawns the `claude` CLI directly.
   - It writes Claude-specific JSONL input to stdin.
   - It parses Claude-specific `stream-json` events from stdout.

2. **Session / backend state model**
   - Session state is shaped around Claude concepts like:
     - `SessionID` / `--resume`
     - `PlanMode`
     - `ActMode`
     - `AskUserQuestion`
     - `EnterPlanMode` / `ExitPlanMode`
     - `/compact`
     - `task_progress` subagent events

3. **GraphQL + frontend UI contract**
   - API names are generic, but the fields and controls are Claude-shaped:
     - `planMode`
     - `actMode`
     - `pendingInteraction`
     - `systemStatus`
     - `subagentActivities`
     - UI buttons for Plan / Act / Compact

---

## 1. Core Claude runtime integration

### `internal/agent/claude.go`
This is the core Claude runner.

#### Process spawning
- `spawnAndRun()` executes:
  - `exec.CommandContext(ctx, "claude", args...)`
- Sets:
  - `cmd.Dir = session.WorkDir`
  - `cmd.Env = buildClaudeEnv()`

#### Claude CLI args
`buildClaudeArgs()` builds Claude-specific startup flags:
- `-p`
- `--verbose`
- `--output-format stream-json`
- `--input-format stream-json`
- `--include-partial-messages`
- `--disallowedTools EnterWorktree ExitWorktree`
- `--dangerously-skip-permissions` when `ActMode`
- `--permission-mode plan` when `PlanMode`
- `--resume <sessionID>` when resuming

This hardcodes Claude-specific assumptions around:
- transport protocol
- permission flags
- tool-disallow mechanism
- resume/session semantics

#### Environment handling
`buildClaudeEnv()` strips `CLAUDECODE=` from the environment to allow nested sessions.

---

## 2. Claude stream protocol parsing

### `internal/agent/parse.go`
This file parses Claude Code's `stream-json` output.

It understands Claude/Anthropic-flavored event shapes such as:
- `stream_event`
- `assistant`
- `content_block_delta`
- `content_block_start`
- `result`
- `error`
- `system`

It also understands Anthropic content block semantics:
- `text`
- `tool_use`
- `input_json_delta`
- `thinking_delta`
- `signature_delta`

It handles Claude-specific system events like:
- `status`
- `task_progress`

So this is not a generic stream parser; it is a Claude-specific parser.

---

## 3. Claude stdin message protocol

### `internal/agent/claude.go` -> `sendToProcess()`
Beans sends user messages to Claude via stdin JSONL in Claude's input format:

- top-level `type: "user"`
- nested `message.role = "user"`
- `message.content` can be text or Anthropic-style content blocks

For images it constructs Anthropic image blocks with base64 sources.

So input submission is also Claude-specific.

---

## 4. Agent manager is generic in name, but hardcodes Claude

### `internal/agent/manager.go`
The manager package is named generically, but new sessions are currently hardcoded to:
- `AgentType: "claude"`

This happens in multiple places:
- `GetSession()` materialization from disk
- `AddInfoMessage()`
- `SetPlanMode()`
- `SetActMode()`
- `SetPendingInteraction()`
- `loadOrCreateSession()`

### `internal/agent/types.go`
The `Session` type includes fields that directly reflect Claude behavior:
- `AgentType string // "claude" for now`
- `SessionID` for CLI `--resume`
- `PlanMode`
- `ActMode`
- `SystemStatus`
- `PendingInteraction`
- `SubagentActivities`

---

## 5. Claude-specific tool and interaction handling

### `internal/agent/claude.go`
The output handling logic has explicit support for Claude tool names and flows.

#### Blocking/special tools
It recognizes and handles:
- `AskUserQuestion`
- `EnterPlanMode`
- `ExitPlanMode`

This feeds into:
- `blockingInteraction()`
- `handleBlockingTool()`
- `autoApproveModeSwitch()`

So the approval/question flow in the UI is currently based on Claude tool names.

#### Plan file convention
`findPlanFilePath()` looks for writes to:
- `~/.claude/plans/*.md`

That means the plan approval flow is partly coupled to Claude file layout.

#### Compact behavior
Beans treats `/compact` as a special user message and uses it to trigger attachment cleanup after the turn completes.

#### Subagent activity
Subagent activity in the UI is derived from Claude `task_progress` events and related stream boundaries.

---

## 6. Separate Claude dependency for workspace descriptions

### `internal/agent/describe.go`
Workspace description generation is a second Claude integration, separate from chat sessions.

It runs:
- `claude --print --model haiku`

This is used to summarize the first user message into a short workspace description.

### `internal/commands/serve.go`
`agentMgr.SetOnFirstUserMessage(...)` calls:
- `agent.GenerateDescription(message)`

So even beyond the main agent runner, Beans depends on Claude for metadata generation.

---

## 7. Server-side prompts assume Claude Code behavior

### `internal/commands/serve.go`
There are two key prompt/context injection paths.

#### `centralAgentPrompt`
This explicitly says:
- do **not** use Claude Code's built-in worktree system
- use the GraphQL `startWork` mutation instead
- use `AskUserQuestion` for user questions

#### Bean/worktree-specific context provider
For worktree agents, Beans injects instructions that:
- forbid Claude Code's built-in worktree system
- require `AskUserQuestion`
- constrain writes to the current worktree

These are not just docs—they reflect real UI/backend assumptions.

---

## 8. GraphQL API is generic in naming, but Claude-shaped in structure

### `internal/graph/schema.graphqls`
Generic names:
- `agentSession`
- `sendAgentMessage`
- `stopAgent`
- `clearAgentSession`

Claude-shaped fields on `AgentSession`:
- `agentType`
- `planMode`
- `actMode`
- `systemStatus`
- `pendingInteraction`
- `subagentActivities`

Claude-shaped mutations:
- `setAgentPlanMode`
- `setAgentActMode`

### `internal/graph/schema.resolvers.go`
When a session is cleared and the subscription needs to emit an empty payload, it constructs a fallback session with:
- `AgentType: "claude"`

### `internal/graph/agent_helpers.go`
Maps the backend session model to GraphQL. The mapper itself is generic, but the underlying fields are Claude-oriented.

---

## 9. Frontend is mostly generic shell, but assumes Claude semantics

### `frontend/src/lib/agentChat.svelte.ts`
Consumes the Claude-shaped GraphQL contract:
- `planMode`
- `actMode`
- `pendingInteraction`
- `subagentActivities`

### `frontend/src/lib/components/AgentChat.svelte`
Assumes:
- there are exactly two modes: `plan` and `act`
- approving plan exit means setting plan=false and act=true
- compacting is done by sending `/compact`

### `frontend/src/lib/components/AgentComposer.svelte`
UI controls are Claude-oriented:
- Plan
- Act
- Compact
- Clear

### `frontend/src/lib/components/PendingInteraction.svelte`
Understands only these interaction types:
- `EXIT_PLAN`
- `ASK_USER`

These map directly to the current Claude integration.

### `frontend/src/lib/thinkingPhrases.ts`
Contains cosmetic Claude-specific copy like:
- "Claude and beans, scheming..."
- "Claude is bean-storming..."

---

## 10. Docs and repo conventions also reference Claude

### `README.md`
Has a dedicated Claude Code section instructing users to add hooks to `.claude/settings.json`.

### Repo-local Claude support files
Additional Claude-specific references exist in:
- `.claude/rules/...`
- `.claude-plugin/...`
- `CLAUDE.md`

These are not part of the web runtime itself, but they are still part of the current Claude-first setup.

---

## 11. Tests encode Claude behavior

Tests that lock in current Claude behavior include:
- `internal/agent/claude_test.go`
- `internal/agent/parse_test.go`
- `internal/agent/manager_test.go`
- `internal/agent/describe_test.go`

They cover things like:
- Claude stream-json parsing
- Claude CLI args
- AskUserQuestion behavior
- EnterPlanMode / ExitPlanMode
- `~/.claude/plans/...`
- `--dangerously-skip-permissions`
- `--permission-mode plan`

---

## Practical architectural takeaway

The main problem is **not only** that Beans spawns `claude`.

The bigger issue is that the current session model and UI contract are shaped around Claude-specific concepts:
- Claude permission modes
- Claude tool names
- Claude stream events
- Claude resume semantics
- Claude compaction command
- Claude subagent progress format

So a future agent abstraction likely needs at least two layers:

1. **runtime/transport adapter**
   - how to start the agent
   - how to send messages
   - how to receive events

2. **capability / UI contract**
   - supports mode switching?
   - supports interactive questions?
   - supports compaction?
   - supports subagent progress?
   - supports resume?

This should help future runs avoid having to rediscover the initial Claude-specific touchpoints from scratch.
