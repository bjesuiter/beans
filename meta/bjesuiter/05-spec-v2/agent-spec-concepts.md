# Beans agent spec v2 concepts

This companion note explains the core concepts used in `./beans-agent-spec-v2.md` in product terms rather than API terms.

It exists to answer questions like:

- What is a **profile**?
- What belongs to Beans policy vs a driver?
- Why do we have both **modes** and **actions**?
- Why are **artifacts** separate from messages?
- Why is workspace description generation outside the live session API?

---

## 1. Driver

A **driver** is the adapter for one concrete runtime or protocol.

Examples:

- Claude CLI
- pi RPC
- Codex MCP
- ACP / OpenCode

A driver is responsible for:

- opening or resuming a live runtime session
- translating Beans input into the runtime's native protocol
- translating runtime events back into Beans events
- reporting runtime-specific host requirements

A driver is **not** responsible for Beans product behavior like:

- deciding what the `__central__` session means
- deciding which prompt policy to use for a worktree chat
- choosing how Beans stores attachments or artifacts
- deciding how the UI renders plan approval or diffs

Short version: a driver adapts a runtime; it does not define Beans product semantics.

---

## 2. Live session

A **live session** is an active conversation with an agent runtime.

It supports operations like:

- send user input
- cancel the current turn
- change mode
- answer a pending interaction
- perform a runtime action like compaction
- close the session

This is the runtime-facing object behind one Beans chat session.

---

## 3. Session state

**Session state** is the canonical Beans-owned view of a session.

It is what the rest of Beans should depend on:

- GraphQL
- subscriptions
- frontend stores
- chat UI
- persistence

The key idea is that Beans should not expose raw Claude events, raw pi packets, or raw ACP payloads directly to the rest of the app.

Instead, drivers emit normalized events and Beans reduces them into one shared session model.

---

## 4. Profile

A **profile** is a Beans-owned policy bundle for a session.

A profile lets Beans inject product-specific instructions and metadata without making those instructions part of the driver contract.

Examples:

- `central_planner`
- `worktree_implementation`

A profile may define things like:

- a system prompt or context prelude
- safety instructions
- workflow conventions
- special product guidance

Examples in Beans:

- the central planner can be told to coordinate work instead of implementing it
- the central planner can be told to use Beans `startWork` instead of runtime-native worktree switching tools
- a worktree implementation session can be told to stay inside the current worktree

Why this matters:

- the behavior is real and important
- but it is **Beans policy**, not a property of Claude, pi, or Codex

So the driver should receive the resulting context, but should not hard-code "if session is central planner, do X".

### Good mental model

Think of a profile as:

- **prompt policy + workflow policy + session metadata**
- chosen by Beans
- consumed by the driver as ordinary input

### Why not put this in the driver?

Because that would make the abstraction leak product semantics downward.

If the driver had built-in knowledge of:

- `__central__`
- `startWork`
- worktree-only safety rules

then the system would still be Claude-shaped or Beans-app-shaped in the wrong layer.

Profiles keep that customization in the correct place.

---

## 5. Mode

A **mode** is a runtime behavior setting exposed by a session.

Examples:

- `act`
- `plan`

Modes are session-level state, not one-off commands.

A runtime may:

- support multiple modes
- support only `act`
- not support mode switching at all

Modes exist because some runtimes have a stable distinction between planning and acting behavior.

But modes should stay generic. The rest of Beans should not depend on Claude flags like:

- `--permission-mode plan`
- `--dangerously-skip-permissions`

Those are driver implementation details.

---

## 6. Interaction

An **interaction** is a structured request from the agent that needs a user response.

Examples:

- confirmation
- select one option
- multi-select
- freeform text input
- editor-style response
- permission approval

The important shift in the spec is:

- interactions are stored in session state
- they are not synchronous host callbacks hidden inside one runtime

That makes them renderable in the UI and answerable through generic Beans APIs.

---

## 7. Action

An **action** is a runtime operation that is neither:

- a normal user message, nor
- a mode switch

Example:

- `compact`

This exists because some runtime behaviors are operational commands rather than conversational turns.

The v2 spec uses actions to replace magic message conventions like sending `/compact` as plain chat text.

### Mode vs action

- **Mode** = persistent session behavior setting
- **Action** = discrete runtime operation

Examples:

- switching from `plan` to `act` is a **mode** change
- compacting the conversation is an **action**

---

## 8. Tool call

A **tool call** is a normalized unit of live work activity.

The spec keeps this concept broad on purpose so it can represent:

- actual tools
- delegated subagents
- background tasks

This is how Beans can preserve the current live activity UI without tying itself to Claude's `task_progress` event format.

---

## 9. Artifact

An **artifact** is a durable session output that should be treated as a first-class object, not just as chat text.

Examples:

- a plan
- a diff summary
- a generated file preview
- a note

Why artifacts exist:

- plans are not just another assistant paragraph
- diffs are not just another tool row
- some outputs should be referenced, reviewed, persisted, and rendered separately

This is especially important for plan approval.

Instead of relying on Claude-specific file discovery like `~/.claude/plans/...`, Beans can work with:

- a `plan` artifact
- an interaction that references that artifact

That makes the workflow runtime-agnostic.

---

## 10. Attachment

An **attachment** is a Beans-owned persisted file referenced by session input or history.

Examples:

- uploaded screenshots
- images attached to a user message

Why attachment handling belongs to Beans:

- Beans persists conversation history
- Beans serves attachment content back to the frontend
- Beans owns cleanup after compaction or clear-session

Drivers may consume attachment bytes or file paths, but they should not own attachment storage policy.

---

## 11. Resume state

**Resume state** is the driver-owned opaque data needed to continue a previous session.

Examples:

- Claude session ID
- pi session identifier
- Codex thread ID
- ACP session ID

Why it is opaque:

- different runtimes resume in different ways
- Beans should store it, route it back to the correct driver, and avoid over-modeling runtime-private details

At the same time, the surrounding persisted envelope is Beans-owned.

So the model is:

- Beans owns the persistence structure
- drivers own the opaque continuation payload inside it

---

## 12. Utility provider

A **utility provider** handles one-off non-session calls.

Current example:

- generate a short workspace description from the first user message

This is intentionally outside the live session API because it is not:

- a persistent conversation
- a streamed turn
- part of durable chat state

The same runtime family might power both live sessions and utility calls, but they are different abstractions and should stay separate.

---

## 13. Layering summary

### Beans owns

- session purpose
- profiles
- prompts and workflow conventions
- canonical session state
- persistence envelopes
- attachments
- artifacts
- UI rendering policy
- GraphQL API shape

### Drivers own

- protocol translation
- runtime process/session lifecycle
- runtime-native resume payloads
- runtime-native mode/action implementation details
- host capability requirements

This is the main architectural point of the v2 spec.

---

## 14. Example: central planner profile

A good example is the central `__central__` session.

What Beans owns:

- deciding that `__central__` is a coordinator session
- attaching a `central_planner` profile
- adding instructions like "use `startWork`"

What the driver owns:

- sending that prompt/context into Claude, pi, or another runtime
- translating runtime output back into Beans events

So the runtime changes, but the Beans product concept stays stable.

---

## 15. Example: plan approval

Another good example is plan approval.

What Beans should see:

- current mode
- a `plan` artifact
- an interaction asking for approval or refinement

What the driver may do internally:

- map that from Claude `ExitPlanMode`
- map that from some pi-specific planning flow
- map that from an ACP-native interaction model

Again, Beans depends on the normalized concept, not the runtime-specific mechanism.

---

## 16. Short glossary

- **Driver**: runtime adapter
- **Live session**: active agent conversation handle
- **Session state**: canonical Beans-owned view of the session
- **Profile**: Beans-owned prompt/workflow policy for a session
- **Mode**: persistent runtime behavior setting
- **Interaction**: structured request awaiting user input
- **Action**: discrete runtime operation like compaction
- **Tool call**: normalized live activity item
- **Artifact**: durable structured session output
- **Attachment**: Beans-owned persisted input file
- **Resume state**: driver-owned opaque continuation payload
- **Utility provider**: helper interface for one-off non-session generation
