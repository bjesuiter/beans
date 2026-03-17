# Use-case: structured user questions

## What it is

Beans supports agent turns that must stop and ask the human for clarification, confirmation, or a choice.

Today this is implemented through Claude's `AskUserQuestion` tool.

## How Beans uses Claude for it

1. Claude begins a tool call named `AskUserQuestion`.
2. Beans does not immediately surface it; it waits until the streamed tool input is complete enough.
3. Beans parses the tool input JSON and extracts structured question data, including:
   - `header`
   - `question`
   - `multiSelect`
   - `options[]`
4. Beans sets a pending interaction on the session:
   - `PendingInteraction.Type = ask_user`
5. Beans signals the Claude process to stop, while keeping the session resumable.
6. The frontend renders a dedicated interaction panel instead of expecting the user to read plain-text questions in chat output.
7. When the user responds, the frontend sends a normal `sendAgentMessage` mutation back to the same session.

## Frontend behavior

The UI currently supports:

- single-select questions
- multi-select questions
- freeform typed responses

Response encoding is intentionally simple:

- single-select -> option label
- multi-select -> selected labels joined by `, `
- typed response -> raw text

There is no separate "answer interaction" RPC.

## Claude-specific behavior in this use-case

- The whole flow depends on the tool name `AskUserQuestion`.
- Beans knows the expected JSON structure of that tool input.
- The web UI assumes that plain-text questions are insufficient and that the Claude tool must be used for interactive prompts.

## Why it matters

This is one of the clearest examples where the UI contract is Claude-shaped. The backend and frontend are both expecting a specific Claude tool and a specific tool-input schema.

## Flow diagram

```mermaid
flowchart TD
    C[Claude invokes AskUserQuestion] --> B[Beans buffers streamed tool input]
    B --> P[Parse structured questions/options]
    P --> I[Set PendingInteraction = ask_user]
    I --> S[Stop Claude process but keep session resumable]
    S --> U[Frontend renders question UI]
    U --> R[User picks option or types reply]
    R --> G[sendAgentMessage]
    G --> X[Resume Claude conversation]
```

## Relevant files

- `internal/agent/claude.go`
- `internal/agent/parse.go`
- `internal/agent/types.go`
- `frontend/src/lib/components/PendingInteraction.svelte`
- `frontend/src/lib/agentChat.svelte.ts`
- `internal/commands/serve.go`

## Assessment against `meta/bjesuiter/05-spec-v2/beans-agent-spec-v2.md`

**Verdict:** directly covered and cleaner than the older spec shape.

Spec v2 handles this use-case very naturally because structured questions become message/frame semantics instead of a dedicated side-channel:

- the runtime emits an `interaction_request` frame
- Beans persists that in the message timeline
- the frontend derives a pending-interaction view from unresolved frames
- the user answer goes back as an outbound message carrying an `interaction_response` frame

That removes the major current Claude coupling points:

- no hard dependency on the tool name `AskUserQuestion`
- no need for a special `Respond(...)` method on the session interface
- no need to overload ordinary plain-text chat as the implicit answer path

It also fits the current Beans UI well because single-select, multi-select, and text responses can all be different payload shapes of the same frame family.

The remaining work is mostly convention-level detail:

- what exact payload schema should `interaction_request` use?
- how should validation or option metadata be encoded?
- which interaction frames should be persisted versus treated as ephemeral?

But at the abstraction level, this use-case is one of the strongest arguments for the message/frame design.