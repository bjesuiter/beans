---
# beans-kf3o
title: Preserve Claude resume and persistence semantics through the driver layer
status: todo
type: task
priority: normal
created_at: 2026-03-17T17:22:08Z
updated_at: 2026-03-17T17:22:37Z
parent: beans-7oo3
---

Ensure the Claude driver keeps the same resume and conversation continuity behavior after the refactor.

## Deliverables

- keep Claude session ID handling intact
- ensure persisted conversation state and resume state stay compatible across restarts
- cover migration edge cases with tests or fixtures
