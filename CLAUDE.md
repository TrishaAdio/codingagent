# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build / Typecheck / Run

```bash
bun run typecheck     # TypeScript type checking (bun x tsc --noEmit)
bun run start         # Run the CLI
bun run dev           # Run with hot-reload (bun --watch)
bun run build         # Production bundle to dist/cli.js
```

Always run `typecheck` after edits. There is no test suite and no linter config in this snapshot.

## Runtime & Module System

- **Bun only** — never use `node` directly. All scripts use `bun`.
- **ESM** (`"type": "module"`), `moduleResolution: "bundler"`.
- **Path alias**: `src/*` maps to `./src/*` (configured in `tsconfig.json` paths). Import as `src/foo/bar.js` (`.js` extension in TS source is conventional in this codebase).
- **JSX**: `react-jsx` transform, components are `.tsx` files in `src/components/`.

## Core Architecture

Three registries power the entire system:

| Registry | File | Purpose |
|----------|------|---------|
| Commands | [src/commands.ts](src/commands.ts) | Slash commands (`/review`, `/commit`, `/compact`, `/config`, etc.) — conditional imports per environment |
| Tools | [src/tools.ts](src/tools.ts) | Agent-callable tools (Bash, Read, Write, Grep, WebFetch, etc.) — permission-aware filtering |
| Skills | [src/skills/](src/skills/) | On-demand workflows loaded from `SKILL.md` files, invoked via `SkillTool` |

Entry point: [src/entrypoints/cli.tsx](src/entrypoints/cli.tsx) — fast-path routing, then delegates to [src/main.tsx](src/main.tsx) (Commander.js CLI parsing + REPL launch).

## Feature Flags (Build-Time Dead Code Elimination)

Feature flags use Bun's `bun:bundle` module for compile-time elimination:

```typescript
import { feature } from 'bun:bundle'

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

- All flags default to `false`. Enable via `FEATURE_FLAGS` env var (comma-separated).
- A dev-time shim lives at [plugins/bunBundleDev.ts](plugins/bunBundleDev.ts), preloaded via `bunfig.toml`.
- Notable flags: `PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `AGENT_TRIGGERS`, `MONITOR_TOOL`, `BG_SESSIONS`, `TEMPLATES`, `BYOC_ENVIRONMENT_RUNNER`, `CHICAGO_MCP`, `DUMP_SYSTEM_PROMPT`, `COMMIT_ATTRIBUTION`, `CONTEXT_COLLAPSE`.

## Build Macros

Defined in [bunfig.toml](bunfig.toml) and injected via Bun's `--define` (also replicated in the `build` script in [package.json](package.json)):

- `MACRO.VERSION` — current version string
- `MACRO.PACKAGE_URL` — npm package name
- `MACRO.FEEDBACK_CHANNEL`, `MACRO.ISSUES_EXPLAINER`, `MACRO.VERSION_CHANGELOG`

## Key Subsystems

- **[src/bridge/](src/bridge/)** — IDE/editor bridge for bidirectional communication between IDE extensions (VS Code, JetBrains) and the CLI. JWT-based auth, session runners, message protocol.
- **[src/services/](src/services/)** — External integrations: MCP server management, OAuth flow, LSP manager, telemetry (OpenTelemetry + gRPC), GrowthBook analytics/feature flags, conversation compaction, policy limits.
- **[src/state/](src/state/)** — Centralized app state using a minimal custom `createStore` pattern (observable with `getState`/`setState`/`subscribe`). Not Redux or Zustand.
- **[src/ink/](src/ink/)** — Customized/forked Ink renderer (~50 files): own reconciler, DOM, layout engine, virtual scrolling, text wrapping, bidi support.
- **[src/coordinator/](src/coordinator/)** — Multi-agent orchestration. `TeamCreateTool`/`TeamDeleteTool` for team-level parallel work.
- **[src/tasks/](src/tasks/)** — Task types: `LocalMainSessionTask`, `LocalShellTask`, `LocalWorkflowTask`, `LocalAgentTask`, `InProcessTeammateTask`, `DreamTask`, `MonitorMcpTask`, `RemoteAgentTask`.
- **[src/hooks/toolPermission/](src/hooks/toolPermission/)** — Permission system: every tool invocation checks permissions. Modes: `default`, `plan`, `bypassPermissions`, `auto`. Uses bash/shell classifiers and dangerous-pattern detection.

## Stubs

[stubs/](stubs/) contains no-op replacements for internal `@ant/*` packages not included in the leak:
- `@ant/claude-for-chrome-mcp`
- `@ant/computer-use-input`
- `@ant/computer-use-mcp`
- `@ant/computer-use-swift`

## Startup Flow

[src/main.tsx](src/main.tsx) parallelizes MDM settings reads, keychain prefetches, and API preconnects as side-effects before heavy module evaluation. Heavy modules (OpenTelemetry ~400KB, gRPC ~700KB) are deferred via dynamic `import()` until needed.

[src/entrypoints/cli.tsx](src/entrypoints/cli.tsx) uses dynamic imports for fast-path routing — e.g., `--version` avoids loading anything beyond the bootstrap.
