# OpenClaw multi-agent runtime notes

## High-level runtime split

OpenClaw supports multiple downstream coding agents via three distinct execution paths:

1. **Embedded Pi runtime**
   - Main/default path for regular model providers.
   - Code path: `src/agents/agent-command.ts` -> `runEmbeddedPiAgent()`.
   - Runtime type in config: `agents.list[].runtime.type = "embedded"`.

2. **Generic CLI backend adapter**
   - Used for CLI-based harnesses that are integrated directly, not through ACP.
   - Code path: `src/agents/agent-command.ts` -> `runCliAgent()` when `isCliProvider(...)`.
   - Built-in backends in `src/agents/cli-backends.ts`:
     - `claude-cli`
     - `codex-cli`
   - Also supports custom configured CLI backends via `agents.defaults.cliBackends`.

3. **ACP runtime**
   - Used for external harness runtimes via an ACP backend plugin.
   - Docs: `docs/tools/acp-agents.md`.
   - Config/runtime types:
     - `agents.list[].runtime.type = "acp"`
     - top-level `acp.backend`
     - per-agent `agents.list[].runtime.acp.*`
     - per-binding `bindings[].type = "acp"`
   - Spawn path: `src/agents/acp-spawn.ts`.

## ACP backend plugin

The default ACP runtime backend appears to be the `acpx` plugin:
- plugin entry: `extensions/acpx/index.ts`
- service registration: `extensions/acpx/src/service.ts`
- runtime: `extensions/acpx/src/runtime.ts`

The `acpx` backend resolves harness commands in:
- `extensions/acpx/src/runtime-internals/mcp-agent-command.ts`

Built-in ACP harness command mapping there:
- `codex` -> `npx @zed-industries/codex-acp`
- `claude` -> `npx -y @zed-industries/claude-agent-acp`
- `gemini` -> `gemini`
- `opencode` -> `npx -y opencode-ai acp`
- `pi` -> `npx pi-acp`

So OpenClaw definitely uses ACP for:
- Codex
- Claude
- Gemini CLI
- OpenCode
- Pi

## Important nuance: OpenCode also exists as a provider catalog

OpenClaw also has bundled provider plugins for:
- `extensions/opencode/index.ts`
- `extensions/opencode-go/index.ts`

Those are **model provider integrations**, not ACP harness runtimes.
So “OpenCode” exists in two different senses in OpenClaw:
1. as a model/API provider catalog (`opencode`, `opencode-go`)
2. as an ACP harness target (`opencode` via `opencode-ai acp`)

## Policy/config hooks

Relevant files:
- `src/config/types.agents.ts`
- `src/config/types.acp.ts`
- `src/acp/policy.ts`
- `src/acp/persistent-bindings.types.ts`

Notable knobs:
- `acp.enabled`
- `acp.dispatch.enabled`
- `acp.backend`
- `acp.defaultAgent`
- `acp.allowedAgents`
- `agents.list[].runtime.type`
- `agents.list[].runtime.acp.agent`
- `agents.list[].runtime.acp.backend`
- `bindings[].type = "acp"`

## Key takeaway for Beans refactor discussion

OpenClaw does **not** force one abstraction path for every downstream agent.
It uses:
- one native embedded runtime,
- one generic CLI adapter family,
- and one ACP runtime family.

That suggests a useful design lesson for Beans:
- don’t assume every future agent should fit a single transport/protocol abstraction immediately;
- instead, consider a higher-level runtime abstraction with multiple backend kinds:
  - embedded/native
  - direct CLI
  - ACP
