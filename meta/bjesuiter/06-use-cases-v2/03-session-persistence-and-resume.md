# Use-case: session persistence and resume

## What it is

Beans keeps agent conversations durable so a bean chat can continue across turns, process restarts, and mode switches.

This is a core Claude-backed use-case because the current implementation depends on Claude session IDs and `--resume`.

## How Beans uses Claude for it

1. Beans stores conversation history in:
   - `.beans/.conversations/<beanID>.jsonl`
2. It appends persisted message entries for user, assistant, tool, and info messages.
3. When Claude emits a `result` event with a `session_id`, Beans stores that value on the session and persists it as a JSONL `meta` entry.
4. On a later turn, if `SessionID` is present, Beans respawns Claude with:
   - `--resume <sessionID>`
5. The resumed Claude process continues the same conversation context instead of starting from scratch.

## Where Beans relies on this

- normal multi-turn chat continuity
- resuming after the process was stopped
- switching modes and restarting the process without losing context
- preserving pending conversations after graceful shutdown

## Claude-specific behavior in this use-case

- `SessionID` is a Claude concept in the current implementation.
- Resume depends on Claude's CLI flag `--resume`.
- Process shutdown tries to be graceful so Claude has a chance to persist resumable state before Beans kills it.
- Session materialization from disk assumes old persisted sessions are Claude sessions.

## Important flow details

### Stop

`StopSession(beanID)` stops the running process but keeps the stored session and conversation history.

### Clear

`ClearSession(beanID)` deletes the JSONL conversation first, then removes the in-memory session, then stops any running process.

### Shutdown

`Manager.Shutdown()` kills all running Claude processes concurrently, but the persistence model is still based on Claude-compatible resume state.

## Flow diagram

```mermaid
flowchart TD
    A[User/agent turn] --> J[Append messages to .beans/.conversations/<beanID>.jsonl]
    J --> R[Claude emits result with session_id]
    R --> M[Persist meta entry with SessionID]
    M --> N[Later message or mode restart]
    N --> X{SessionID present?}
    X -->|Yes| C[Respawn claude with --resume <sessionID>]
    X -->|No| F[Start fresh Claude session]
    C --> H[Conversation context continues]
    F --> H
```

## Relevant files

- `internal/agent/manager.go`
- `internal/agent/store.go`
- `internal/agent/claude.go`
- `internal/agent/types.go`
- `internal/graph/schema.resolvers.go`

## Assessment against `meta/bjesuiter/05-spec-v2/beans-agent-spec-v2.md`

**Verdict:** directly solved and one of the strongest fits in v2.

This use-case maps very naturally onto the reduced message/frame model:

- Beans persists durable `Messages` as the canonical session timeline
- driver-specific continuation data lives in opaque `ResumeToken`
- `Resume(...)` is a first-class primitive on the agent interface
- Beans no longer has to model persistence around Claude-specific session concepts

This is especially important because different runtimes resume differently:

- Claude may use a resumable session ID
- pi may use a session handle or file
- Codex may use a thread ID
- ACP may use an ACP session identifier

V2 also improves the separation of concerns:

- Beans owns the persisted timeline format
- the driver owns the meaning of the resume token
- the reducer can rebuild derived views from messages/frames without depending on a Claude-shaped in-memory session

The remaining work is mostly implementation detail:

- exact JSONL/on-disk format
- migration of old Claude-only persisted sessions
- how aggressively to persist incremental frames versus normalized final message content

So this is not just simplified in v2; it is one of the clearest examples of the new abstraction doing exactly the right thing.