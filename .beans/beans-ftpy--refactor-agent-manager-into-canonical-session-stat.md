---
# beans-ftpy
title: Refactor agent manager into canonical session-state owner
status: todo
type: task
priority: normal
created_at: 2026-03-17T17:21:56Z
updated_at: 2026-03-17T17:22:37Z
parent: beans-fwpc
---

Restructure internal/agent/Manager so it owns canonical state and driver lifecycle management.

## Deliverables

- manager reduces normalized events into session state
- manager owns live-session open/send/cancel/respond/close orchestration
- subscriptions publish from the unified state store
