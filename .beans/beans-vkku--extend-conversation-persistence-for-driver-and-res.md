---
# beans-vkku
title: Extend conversation persistence for driver and resume metadata
status: todo
type: task
priority: normal
created_at: 2026-03-17T17:21:56Z
updated_at: 2026-03-17T17:22:37Z
parent: beans-fwpc
---

Evolve the session persistence format without breaking existing conversation files.

## Deliverables

- read current JSONL conversations compatibly
- add driver-kind and resume metadata entries
- migrate lazily on write with tests
