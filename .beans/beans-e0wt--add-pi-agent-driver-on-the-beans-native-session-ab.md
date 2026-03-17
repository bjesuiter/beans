---
# beans-e0wt
title: Add pi agent driver on the Beans-native session abstraction
status: todo
type: epic
priority: normal
created_at: 2026-03-17T17:12:58Z
updated_at: 2026-03-17T17:13:08Z
parent: beans-q1ax
blocked_by:
    - beans-fwpc
---

Add an initial pi RPC-backed driver using the new Beans-owned session model and GraphQL/UI flow.

## Scope

- [ ] Implement a pi driver that opens, streams, and resumes sessions
- [ ] Map pi behavior onto Beans messages, status, resume state, and interactions
- [ ] Expose act-only mode and correct capability flags
- [ ] Reuse the existing GraphQL/UI session flow without a second frontend path
- [ ] Add integration tests for the pi-backed flow
