---
# beans-p8jt
title: Normalize Claude streaming output into agent driver events
status: todo
type: task
priority: normal
created_at: 2026-03-17T17:22:08Z
updated_at: 2026-03-17T17:22:37Z
parent: beans-7oo3
---

Extract a Claude event adapter that turns the current stream-json output into normalized Beans events.

## Deliverables

- map message, tool, interaction, status, error, and end-of-session events
- preserve existing Claude parsing behavior while changing the output shape
