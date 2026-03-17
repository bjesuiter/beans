# Beans agent refactor implementation plan

Spec: `meta/bjesuiter/03-spec/beans-agent-spec.md`

## Non-negotiable constraint

**All current functionality must persist during this refactor.**

## Decision

Use a **strangler refactor**:

- keep the current Claude-backed behavior intact
- introduce the Beans-native abstraction underneath it
- make Claude the first driver on that abstraction
- keep the existing GraphQL and frontend contract until feature parity is proven

We will **not**:

- do a full replacement first
- build a second parallel frontend/storage flow

## Why

- A direct replacement is too risky while current behavior must keep working.
- A second flow would create long-term duplication in session state, storage, and UI logic.
- The safest path is one new internal model with compatibility shims at the API boundary.

## Delivery structure

### Milestone

- `beans-q1ax` — **Beans-native multi-agent session refactor**

### Epics

- `beans-fwpc` — **Refactor agent manager to Beans-native driver abstraction**
- `beans-7oo3` — **Adapt Claude implementation to the new driver API**
- `beans-e0wt` — **Add pi agent driver on the Beans-native session abstraction**

Dependency notes:

- `beans-7oo3` is blocked by `beans-fwpc`
- `beans-e0wt` is blocked by `beans-fwpc`

## Phases

### 1. Lock down current behavior

Before changing architecture, add or tighten tests around:

- message streaming
- resume/session persistence
- plan/act switching
- pending interaction approval flow
- stop/clear session
- image attachments
- subagent activity updates

Primary bean: `beans-fwpc`

### 2. Introduce the core abstraction

Add Beans-native core types matching the spec:

- `Driver`
- `LiveSession`
- `Event`
- `SessionState`
- reducer/event application layer

Primary bean: `beans-fwpc`

### 3. Turn the manager into the canonical state owner

Refactor `internal/agent/Manager` to:

- own canonical session state
- reduce normalized driver events into that state
- publish subscriptions from that single state store
- remain the GraphQL entrypoint

Primary bean: `beans-fwpc`

### 4. Wrap Claude as the first driver

Convert the current Claude integration into a `ClaudeDriver` that:

- reuses existing process spawning and parsing logic
- emits normalized Beans events
- preserves current resume behavior
- maps current blocking flows into normalized interactions

Primary bean: `beans-7oo3`

### 5. Keep GraphQL/frontend compatibility during migration

Maintain the current UI contract initially, deriving legacy fields from the new state:

- `planMode` / `actMode` from `CurrentModeID`
- `pendingInteraction` from normalized pending requests
- existing messages/status from canonical session state

Primary bean: `beans-fwpc`

### 6. Extend persistence without breaking existing data

Keep `.beans/.conversations/*.jsonl` working.

Approach:

- read old conversation files as Claude sessions by default
- add versioned/meta entries for driver kind and resume state
- migrate lazily on write, not via a big-bang rewrite

Primary beans: `beans-fwpc`, `beans-7oo3`

### 7. Add the pi driver

After Claude parity is established, add a pi driver that:

- uses the same manager/state/reducer path
- exposes act-only mode initially
- reuses the existing GraphQL/UI flow
- avoids any second frontend or storage path

Primary bean: `beans-e0wt`

## Success criteria

- current Claude functionality remains intact
- only one UI-facing session model exists
- only one persistence flow exists
- Claude runs through the new driver abstraction with parity
- pi can be added without further frontend architecture changes
