---
# beans-q1ax
title: Beans-native multi-agent session refactor
status: todo
type: milestone
created_at: 2026-03-17T17:12:58Z
updated_at: 2026-03-17T17:12:58Z
---

Implement the Beans-native agent abstraction from meta/bjesuiter/04-spec/beans-agent-spec.md while preserving all current Claude-backed functionality.

## Goals

- [ ] Preserve current agent behavior end to end
- [ ] Introduce a single Beans-owned session model
- [ ] Refactor the current Claude implementation behind a driver abstraction
- [ ] Add a pi agent driver on the new abstraction
- [ ] Keep GraphQL/UI compatible during migration
