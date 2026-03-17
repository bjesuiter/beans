# Claude integration use-cases in Beans, reassessed for spec v2

This directory copies the concrete Beans use-cases that currently depend on the Claude integration described in `meta/bjesuiter/01-research/claude-code-touchpoints.md`.

Unlike `../04-use-cases/`, these copies are reassessed against `meta/bjesuiter/05-spec-v2/beans-agent-spec-v2.md`.

## Use-cases

1. [`01-central-planning-agent.md`](./01-central-planning-agent.md)
   - The central `__central__` agent that plans work, manages beans, and starts worktrees.
2. [`02-worktree-implementation-agent.md`](./02-worktree-implementation-agent.md)
   - Bean-specific agent sessions that do implementation work inside a managed worktree.
3. [`03-session-persistence-and-resume.md`](./03-session-persistence-and-resume.md)
   - Persisting chat history and Claude session IDs so Beans can resume work.
4. [`04-multimodal-message-input.md`](./04-multimodal-message-input.md)
   - Sending normal user messages and image attachments into Claude.
5. [`05-structured-user-questions.md`](./05-structured-user-questions.md)
   - Handling `AskUserQuestion` so the web UI can show interactive prompts.
6. [`06-plan-mode-and-plan-approval.md`](./06-plan-mode-and-plan-approval.md)
   - Switching between plan and act behavior, including `EnterPlanMode` / `ExitPlanMode`.
7. [`07-conversation-compaction.md`](./07-conversation-compaction.md)
   - Triggering Claude compaction with `/compact` and cleaning up attachments afterwards.
8. [`08-subagent-activity-and-system-status.md`](./08-subagent-activity-and-system-status.md)
   - Surfacing Claude system status and `task_progress` activity in the UI.
9. [`09-workspace-description-generation.md`](./09-workspace-description-generation.md)
   - Running a separate lightweight Claude call to generate short workspace descriptions.

## Cross-cutting takeaway

Even though Beans exposes mostly generic names like `agentSession`, `sendAgentMessage`, and `AgentChat`, the current implementation is still Claude-shaped in three important ways:

- **Runtime:** Beans spawns the `claude` CLI and speaks Claude `stream-json` on stdin/stdout.
- **State model:** session state includes Claude concepts like `SessionID`, plan/act modes, pending interactions, and subagent activity.
- **Frontend contract:** the UI is built around Claude behaviors like `AskUserQuestion`, plan approval, and `/compact`.
