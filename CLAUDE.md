# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working on code in this repository.

## Project Overview

Context Lens is a local observability tool for AI coding agents. It consists of:
1. **Proxy Server** (`:4040`): Intercepts LLM API calls (Anthropic, OpenAI, Gemini, etc.) and writes captures to disk.
2. **Analysis Server** (`:4041`): Reads captures, parses requests, estimates tokens/costs, computes context composition, and serves the Web UI.
3. **CLI**: Manages the proxy lifecycle, spawns tools with correct environment variables, and provides utility commands.

## Common Commands

```bash
# Install dependencies
pnpm install

# Build the project (generates types, compiles TS to dist/)
pnpm run build

# Run the CLI (requires build first)
pnpm run start

# Run tests (compiles to dist-test/ then runs via node:test)
pnpm run test

# Lint and format
pnpm run lint
pnpm run lint:fix

# Dev mode (watch TS compilation)
pnpm run dev

# Build the web UI (from ui/ directory)
pnpm run build:ui
```

## Architecture

The project is split into three main parts:

### 1. Proxy Server (`src/proxy/`)
- **`server.ts`**: Entry point for the proxy server (port 4040).
- **`capture.ts`**: Writes request/response pairs to disk as JSON files.
- **`config.ts`**: Configuration loading (ports, privacy levels).

### 2. Analysis Server (`src/analysis/` + `src/server/`)
- **`server.ts`**: Entry point for the analysis server (port 4041).
- **`watcher.ts`**: Watches the capture directory for new files.
- **`ingest.ts`**: Parses captures and updates the state store.
- **`store.ts`**: Manages persistent state (`state.jsonl`, `.lhar` files).
- **`api.ts`**: REST API endpoints for the web UI.
- **`webui.ts`**: Serves the static web UI bundle.

### 3. Core Logic (`src/core/`)
- **`parse.ts`**: Parses LLM request/response bodies.
- **`routing.ts`**: Detects provider (Anthropic, OpenAI, Gemini) and routes requests.
- **`source.ts`**: Detects the source tool (Claude Code, Codex, Aider, etc.).
- **`tokens.ts`**: Estimates token counts and costs.
- **`session-format.ts`**: Groups API calls into sessions/conversations.
- **`health.ts`**: Computes context health scores and findings.

### 4. CLI (`src/cli.ts` + `src/cli-utils.ts`)
- **`cli.ts`**: Main entry point for the `context-lens` command.
- **`cli-utils.ts`**: Parses arguments, manages tool configurations, and spawns child processes.

### 5. Web UI (`ui/`)
- **Vue 3 + Vite** application.
- **`src/stores/`**: Pinia stores for state management.
- **`src/components/`**: Reusable UI components.

## Key Files and Directories

- `src/proxy/` - Reverse proxy and capture logic.
- `src/analysis/` - Analysis server and ingest logic.
- `src/server/` - API, store, and web UI serving.
- `src/core/` - Shared domain logic (parsing, routing, tokens, sessions).
- `src/lhar/` - LHAR (LLM HTTP Archive) export logic.
- `test/` - Node:test suite (compiled to `dist-test/`).
- `ui/` - Frontend application (Vue 3 + Vite).
- `mitm_addon.py` - Mitmproxy addon for HTTPS interception (used by Codex, OpenCode).
- `schema/` - LHAR JSON schema and generated TypeScript types.

## Development Conventions

- **TypeScript**: ESM project (`"type": "module"`), strict mode enabled, Node16 module resolution.
- **Linting**: Biome is used for formatting and linting. Indentation is 2 spaces.
- **Tests**: Node:test runner. Tests are compiled to `dist-test/` and executed via `node --test`.
- **Generated Files**: `src/lhar-types.generated.ts` and `src/version.generated.ts` are auto-generated. Do not edit manually.

## Adding a New Tool Integration

To add support for a new AI coding tool:

1. **Provider Detection** (`src/core/routing.ts`): If the tool uses a new API format, add a detection rule in `detectProvider()`.
2. **Source Detection** (`src/core/source.ts`): Add an entry to `HEADER_SIGNATURES` or `SOURCE_SIGNATURES` so the UI can label requests.
3. **CLI Integration** (`src/cli-utils.ts`): Add a tool config to `TOOL_CONFIG` with the appropriate environment variables (e.g., `ANTHROPIC_BASE_URL`, `OPENAI_BASE_URL`).
4. **Tests**: Add tests in `test/` for the new detection logic.

## Performance Considerations

- **Startup Time**: The analysis server loads `state.jsonl` on startup. Keep `Store.loadState()` and migrations efficient.
- **Disk I/O**: Avoid reading detail files (`details/*.json`) inside hot loops. Use marker files to skip completed migrations.
- **State Rewrites**: `saveState()` rewrites the entire file. Use `appendToState()` for new entries. Only call `saveState()` after structural changes (eviction, deletion, migrations).

## Data Persistence

- **Default Location**: `~/.context-lens/data/`
- **Files**:
  - `state.jsonl`: Persistent state across restarts.
  - `*.lhar`: Per-session LHAR files.
  - `details/*.json`: Detailed request/response data.

## Related Documentation

- `README.md`: User-facing documentation and usage examples.
- `CONTRIBUTING.md`: Detailed contribution guide, architecture notes, and release process.
- `AGENTS.md`: Agent-specific instructions and development conventions.
- `docs/server-structure.md`: Mermaid diagram of server architecture.
