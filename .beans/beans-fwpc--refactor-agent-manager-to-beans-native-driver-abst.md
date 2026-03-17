---
# beans-fwpc
title: Refactor agent manager to Beans-native driver abstraction
status: todo
type: epic
priority: normal
created_at: 2026-03-17T17:12:59Z
updated_at: 2026-03-17T17:13:08Z
parent: beans-q1ax
---

Restructure the current agent stack so Beans owns canonical session state, event reduction, persistence, and GraphQL mapping without regressing current behavior.

## Scope

- [ ] Lock current behavior with tests
- [ ] Add core driver/session/event/state interfaces
- [ ] Move manager to canonical reducer/state owner
- [ ] Keep existing GraphQL/frontend behavior via compatibility mapping
- [ ] Extend persistence to support driver/resume metadata compatibly
