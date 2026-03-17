---
# beans-kzzy
title: Implement ClaudeDriver live-session lifecycle
status: todo
type: task
priority: normal
created_at: 2026-03-17T17:22:08Z
updated_at: 2026-03-17T17:22:37Z
parent: beans-7oo3
---

Wrap Claude process orchestration behind the new Driver and LiveSession interfaces.

## Deliverables

- implement open/resume/send/cancel/respond/close for Claude-backed sessions
- keep process lifecycle and streaming behavior feature-parity with the current integration
