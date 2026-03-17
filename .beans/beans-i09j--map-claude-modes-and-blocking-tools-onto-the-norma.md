---
# beans-i09j
title: Map Claude modes and blocking tools onto the normalized interaction model
status: todo
type: task
priority: normal
created_at: 2026-03-17T17:22:08Z
updated_at: 2026-03-17T17:22:37Z
parent: beans-7oo3
---

Preserve Claude-specific plan/act semantics using the new session abstraction.

## Deliverables

- expose Claude modes through CurrentModeID and AvailableModes
- map blocking tools and approvals into normalized pending interaction state
- preserve current auto-approval and resume edge cases
