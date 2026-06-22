# Project Guidelines

## Build and Test

```bash
npm install --legacy-peer-deps   # install dependencies
bun run typecheck                # TypeScript type checking (bun x tsc --noEmit)
bun run start                    # run CLI
bun run dev                      # run with hot-reload (--watch)
bun run build                    # production bundle
```

There is no test suite in this snapshot.

## Architecture

Claude Code is a **Bun-first TypeScript CLI** with a **React + Ink** terminal UI. The entry point is `src/entrypoints/cli.tsx`, which bootstraps and delegates to `src/main.tsx` (Commander.js-based argument parsing and REPL initialization).

Three core registries power the system:
- **Commands** (`src/commands.ts`) — slash commands (`/review`, `/init`, `/commit`, etc.)
- **Tools** (`src/tools.ts`) — agent-callable tools (read, write, bash, search, etc.)
- **Skills** (`src/skills/`) — on-demand workflows loaded from `SKILL.md` files

Key subsystems:
- `src/bridge/` — IDE/editor bridge for remote control
- `src/context/` — system/user context assembly
- `src/services/` — external integrations (MCP, telemetry, OAuth, analytics)
- `src/plugins/` — plugin discovery and loading
- `src/state/` — centralized app state (`createStore` pattern)

## Conventions

- **Runtime**: Bun only. Use `bun` for all scripts, not `node`.
- **Path aliases**: `src/*` maps to `./src/*`. Import as `src/foo/bar.js` (note `.js` extension in TS source is common).
- **JSX**: `react-jsx` transform, components are `.tsx` in `src/components/`.
- **Feature flags**: Gated with `feature('FLAG_NAME')` from `bun:bundle`. Set via `FEATURE_FLAGS` env var (comma-separated). Defaults are `false`.
- **Build macros**: `MACRO.VERSION`, `MACRO.PACKAGE_URL`, etc. defined in `bunfig.toml`, injected by Bun's `--define`.
- **Module system**: ESM (`"type": "module"`), `moduleResolution: "bundler"`.
- **Stubs**: `stubs/` contains no-op replacements for internal Anthropic packages not included in the leak.

## Key Files

| File | Purpose |
|------|---------|
| `src/main.tsx` | Full CLI setup, Commander options, REPL launch |
| `src/entrypoints/cli.tsx` | Bootstrap entrypoint with fast-path routing |
| `src/commands.ts` | Command registry and conditional imports |
| `src/tools.ts` | Tool registry with permission-aware filtering |
| `src/Tool.ts` | Tool type definitions and lifecycle |
| `src/QueryEngine.ts` | Core Anthropic API caller |
| `bunfig.toml` | Bun config, macro defines, preload plugin |
| `plugins/bunBundleDev.ts` | Shim for `bun:bundle` feature flag system |

See `README.md` for setup instructions, environment variables, and the leak context.
