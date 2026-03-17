---
# beans-7oo3
title: Adapt Claude implementation to the new driver API
status: todo
type: epic
priority: normal
created_at: 2026-03-17T17:12:59Z
updated_at: 2026-03-17T17:13:08Z
parent: beans-q1ax
blocked_by:
    - beans-fwpc
---

Wrap the current Claude Code process integration behind the new driver interface while keeping full feature parity.

## Scope

- [ ] Convert Claude streaming/parser logic into normalized driver events
- [ ] Preserve resume/session persistence behavior
- [ ] Preserve plan/act mode behavior and interaction handling
- [ ] Preserve tool call, subagent, attachment, and error reporting behavior
- [ ] Verify parity with tests and migration fixtures
