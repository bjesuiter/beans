# Use-case: multimodal message input

## What it is

Beans lets the user send plain text prompts and optional image attachments to the agent chat.

This currently works through Claude's input protocol.

## How Beans uses Claude for it

1. The frontend calls `sendAgentMessage(beanId, message, images)`.
2. If images are present, Beans saves them under:
   - `.beans/.conversations/attachments/<beanID>/<uuid>.<ext>`
3. Beans appends the user message to session history and persists it.
4. If Claude is already running, Beans writes the new message to the current process stdin.
5. Otherwise, Beans spawns a new Claude process and sends the message as the initial prompt.
6. Beans writes one JSON object per line using Claude's `stream-json` input format.

## Message shapes

### Text only

```json
{
  "type": "user",
  "message": {
    "role": "user",
    "content": "plain text"
  }
}
```

### Text plus images

When images are attached, Beans switches to Anthropic-style content blocks and base64-inlines the images:

```json
{
  "type": "user",
  "message": {
    "role": "user",
    "content": [
      { "type": "text", "text": "Please inspect this screenshot" },
      {
        "type": "image",
        "source": {
          "type": "base64",
          "media_type": "image/png",
          "data": "..."
        }
      }
    ]
  }
}
```

## Claude-specific behavior in this use-case

- Beans always writes Claude-compatible `type: "user"` JSONL messages.
- Attached images are encoded in Anthropic content-block format.
- Sending follow-up messages to an already-running process depends on Claude's stream interleaving behavior.

## Constraints

- supported image types: JPEG, PNG, GIF, WebP
- max attachment size: 5 MB
- attachments are later served back to the frontend from `/api/attachments/<beanID>/<imageID>`

## Flow diagram

```mermaid
flowchart TD
    U[User sends text/images] --> G[sendAgentMessage]
    G --> A[Save attachments to .beans/.conversations/attachments]
    A --> S[Append user message to session and JSONL]
    S --> F{Claude process running?}
    F -->|Yes| I[Write stream-json user message to stdin]
    F -->|No| C[Spawn claude and send initial message]
    I --> P[Claude receives text or text+image blocks]
    C --> P
```

## Relevant files

- `internal/agent/manager.go`
- `internal/agent/claude.go`
- `internal/agent/store.go`
- `frontend/src/lib/agentChat.svelte.ts`
