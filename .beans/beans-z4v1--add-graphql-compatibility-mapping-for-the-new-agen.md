---
# beans-z4v1
title: Add GraphQL compatibility mapping for the new agent session model
status: todo
type: task
priority: normal
created_at: 2026-03-17T17:21:56Z
updated_at: 2026-03-17T17:22:37Z
parent: beans-fwpc
---

Keep the existing GraphQL and frontend contract working while the backend migrates to the new session model.

## Deliverables

- derive legacy fields such as planMode, actMode, and pendingInteraction from canonical state
- preserve current subscription and query behavior
