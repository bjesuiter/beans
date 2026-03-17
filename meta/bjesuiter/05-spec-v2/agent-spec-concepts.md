# Beans agent spec v2 concepts

This note explains the reduced concept model behind `./beans-agent-spec-v2.md`.

The main question is:

> What are the **absolute primitives** of the AbstractAgent interface if we want minimal API surface without losing expressability?

The answer in v2 is:

- **Agent**
- **Session**
- **Message**
- **Frame**
- **ResumeToken**
- optional **OneShotCall**

Everything else should either be:

- a frame semantic
- a Beans-owned policy
- or a derived view over persisted messages

---

## 1. Why move to messages + frames?

Beans has to persist conversation history somewhere.
That means the abstraction needs a durable unit.

That durable unit is the **message**.

At the same time, runtimes stream partial and structured output:

- text deltas
- tool activity
- questions for the user
- mode changes
- actions
- status updates
- artifact references

That streaming/update unit is the **frame**.

So the model becomes:

- **messages are durable**
- **frames are expressive**

This gives Beans both:

- a clean persisted timeline
- a rich streaming protocol surface

without requiring a large set of top-level interface methods.

---

## 2. Primitive: Agent

An **Agent** is the runtime adapter.

Examples:

- Claude adapter
- pi adapter
- Codex adapter
- ACP / OpenCode adapter

Its job is small:

- open a session
- resume a session
- optionally handle one-shot work

It should not define Beans product semantics.

So an agent should **not** own concepts like:

- central planner behavior
- worktree workflow policy
- how Beans stores attachments
- how Beans renders plan approval

Those belong to Beans.

---

## 3. Primitive: Session

A **Session** is a live connection to one runtime conversation.

Its job is also small:

- accept outbound messages
- emit inbound frame events
- close

That is why v2 removes separate top-level methods like:

- `SetMode(...)`
- `Respond(...)`
- `PerformAction(...)`
- `Cancel(...)`

All of those are expressible as outbound messages with specific frame semantics.

---

## 4. Primitive: Message

A **Message** is the durable timeline entry that Beans persists.

Examples:

- a user prompt
- an assistant response
- a control message
- a system note

A message is not just plain text.
It is a container for frames.

That matters because many important Beans behaviors are not pure text:

- tool calls
- interaction requests
- interaction responses
- plan artifacts
- mode changes
- control actions like compact

Those all belong in the durable timeline too.

So instead of inventing many special top-level objects, v2 treats them as structured message content.

---

## 5. Primitive: Frame

A **Frame** is the atomic expressive unit.

A frame can represent:

- content
- control
- structure
- state updates
- metadata

Examples:

- `text`
- `attachment`
- `tool_call`
- `interaction_request`
- `interaction_response`
- `mode_change`
- `action`
- `artifact`
- `status`
- `error`

This is the key simplification.

Instead of saying:

- tool calls are one API family
- interactions are another API family
- actions are another API family
- modes are another API family

v2 says:

- they are all just **frame types**

That keeps the interface small while keeping the model rich.

### Durable vs ephemeral frames

Not every frame needs to be persisted.

Persistable examples:

- final text
- interaction requests and answers
- action invocations
- artifact references
- meaningful tool results

Ephemeral examples:

- typing pulses
- transient progress ticks
- raw deltas later folded into final text

So frames are the expressive primitive, but Beans still decides what belongs in durable history.

---

## 6. Primitive: ResumeToken

A **ResumeToken** is the opaque driver-owned value needed to continue a session.

Examples:

- Claude session ID
- pi session handle
- Codex thread ID
- ACP session ID

Beans owns the surrounding persistence structure.
The driver owns the meaning of the token.

This lets Beans support multiple runtimes without pretending they all resume the same way.

---

## 7. Primitive: OneShotCall

A **OneShotCall** is optional and handles work outside the durable session timeline.

Examples:

- generate a workspace description from the first user message
- generate a short session name
- create some metadata summary

This replaces the heavier old idea of a separate "utility provider" concept.

Important nuance:

- a one-shot call is separate from a live session abstraction
- but it may still reuse the same runtime adapter underneath

That means Beans can reuse the user's existing subscription/payment path for helper tasks, without forcing those tasks into chat persistence.

---

## 8. What is no longer a primitive?

These concepts are still useful, but they should no longer enlarge the AbstractAgent method surface.

### Profile
A Beans-owned way to compile prompts and workflow conventions into open context.
Not an agent primitive.

### Mode
Not a special session method.
Just a frame semantic such as `mode_change`.

### Action
Not a special session method.
Just a frame semantic such as `action`.

### Interaction
Not a special reply API.
Just frames:

- `interaction_request`
- `interaction_response`

### Tool call
Not a separate abstract data model.
Just a frame semantic like `tool_call`.

### Artifact
Not a separate top-level protocol family.
Just a frame semantic like `artifact`, plus Beans-side derived views.

### Attachment
Not an abstract-agent primitive.
It is Beans-owned persisted storage referenced by `attachment` frames.

### Status rows / subagent activity
Not abstract primitives.
They are live derived views over `status` and `tool_call` frames.

---

## 9. Reduction map

Here is the v2 reduction more directly.

### Before
Possible separate concepts:

- driver
- session
- profile
- mode
- interaction
- action
- tool call
- artifact
- attachment
- utility provider
- resume state

### After
Interface primitives:

- driver
- session
- message
- frame
- resume token
- optional one-shot call

Everything else becomes:

- frame semantics
- Beans policy
- or derived state

That is the core simplification.

---

## 10. Why this does not lose expressability

This model is smaller, but not weaker.

It can still express:

- normal chat text
- multimodal input
- Claude-style tool calls
- AskUserQuestion-style user prompts
- plan/act transitions
- compaction
- plan artifacts
- diff artifacts
- subagent activity
- helper calls for naming and descriptions

The trick is that expressability moves into:

- frame types
- frame payloads
- reduction rules

instead of moving into:

- more interface methods
- more top-level runtime concepts

So the API surface stays small while the payload vocabulary stays rich.

---

## 11. Examples

### Tool call as message content

A tool call is just a message carrying `tool_call` frames.

```json
{
  "id": "msg_tool_1",
  "role": "assistant",
  "frames": [
    { "type": "tool_call", "phase": "start", "data": { "toolName": "Write" } },
    { "type": "tool_call", "phase": "point", "data": { "path": "foo.go" } },
    { "type": "tool_call", "phase": "end", "data": { "status": "success" } }
  ]
}
```

### Interaction as message content

```json
{
  "id": "msg_interaction_1",
  "role": "assistant",
  "frames": [
    {
      "type": "interaction_request",
      "phase": "point",
      "data": {
        "requestId": "req_1",
        "kind": "select",
        "title": "Choose next step",
        "options": ["Implement", "Refine plan"]
      }
    }
  ]
}
```

### Control operation as message content

```json
{
  "id": "msg_control_1",
  "role": "control",
  "frames": [
    { "type": "action", "phase": "point", "data": { "actionId": "compact" } }
  ]
}
```

This is the whole v2 idea in practice.

---

## 12. Beans-owned policy layer

The fact that the abstract interface is smaller does **not** mean Beans becomes generic and policy-free.

Beans still owns:

- choosing central planner vs worktree context
- building prompts/context frames
- deciding when to use `startWork`
- storing and serving attachments
- deciding which frames are persisted
- deriving UI views like pending interactions and artifact lists

So v2 reduces the runtime interface, not the Beans product.

---

## 13. Suggested mental split

If you need a compact mental model, use this:

### Runtime primitives
- Agent
- Session
- ResumeToken
- OneShotCall

### Timeline primitives
- Message
- Frame

### Beans policy / derivation
- profiles
- persistence rules
- UI views
- workflow conventions

This is probably the smallest concept set that still fits Beans well.

---

## 14. Short glossary

- **Agent**: runtime adapter
- **Session**: live runtime conversation handle
- **Message**: durable timeline entry
- **Frame**: atomic expressive content/update unit
- **ResumeToken**: opaque continuation payload
- **OneShotCall**: optional helper invocation outside durable session history

Non-primitives in v2:

- **Profile**: Beans-owned prompt/workflow policy
- **Mode**: a frame semantic
- **Action**: a frame semantic
- **Interaction**: a pair of frame semantics
- **Tool call**: a frame semantic
- **Artifact**: a frame semantic plus derived view
- **Attachment**: Beans-owned storage referenced by frames

---

## 15. Final rule of thumb

If a new feature can be expressed by adding:

- a new frame type,
- a new frame payload,
- or a new Beans-side derived view,

then it should usually **not** expand the AbstractAgent interface.

That is the reduction principle behind v2.