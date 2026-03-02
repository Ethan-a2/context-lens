# Context Lens — Agent Instructions

## Project overview

Context Lens is a local observability tool for AI coding agents. It runs a **local reverse proxy** that captures LLM HTTP requests/responses (Anthropic/OpenAI/Gemini/OpenAI-compatible, plus MITM mode for Cloudflare-protected traffic) and an **analysis server + web UI** that:

- parses captures into sessions/conversations
- estimates tokens and costs
- computes “context composition” (system/tool defs/history/tool results/thinking/images)
- produces findings (large tool results, overflow risk, unused tool defs, compaction events)
- supports LHAR export (LLM HTTP Archive)

Primary technologies:

- **Node.js + TypeScript** (ESM, Node16 module resolution)
- **Hono** (`hono`, `@hono/node-server`) for HTTP server/API
- **Biome** for lint/format
- **node:test** for tests (compiled to `dist-test/`)
- **Vite + Vitest** for the web UI (in `ui/`)
- **Python**: optional `mitmproxy` addon (`mitm_addon.py`) for HTTPS interception

High-level layout:

- `src/cli.ts`, `src/cli-utils.ts` — CLI wrapper (sets env vars, starts/stops proxy/UI, spawns tools)
- `src/proxy/*` — reverse proxy + capture
- `src/analysis/*` — analysis server/ingest/watch
- `src/server/*` — API, store, projections, web UI serving
- `src/core/*` — parsing, routing/provider detection, security, tokenization, session analysis
- `schema/` — LHAR JSON schema and generated TS types
- `test/` — node:test suite
- `ui/` — frontend bundle (Vite)

## How to build, run, and test

### Install dependencies

```bash
pnpm install
```

### Build

Builds version + schema types, then runs TypeScript compilation to `dist/`.

```bash
pnpm run build
```

### Run locally

Runs the built CLI entrypoint.

```bash
pnpm run build
pnpm run start
```

Typical usage (spawns a tool with env vars pointed at the proxy):

```bash
context-lens claude
context-lens codex
context-lens gemini
context-lens aider --model claude-sonnet-4
context-lens pi
context-lens -- python my_agent.py
```

Ports (defaults from docs):

- Proxy: `:4040`
- Analysis/UI: `:4041` (Web UI at http://localhost:4041)

### Tests

This project compiles tests to `dist-test/` and runs them with Node’s test runner.

```bash
pnpm run test
```

Useful subcommands:

```bash
pnpm run build:test
```

### Lint / format

```bash
pnpm run lint
pnpm run lint:fix
```

### Web UI build

The release pipeline builds the UI from `ui/`.

```bash
pnpm run build:ui
```

## Development conventions

### TypeScript / module system

- ESM project (`"type": "module"` in root `package.json`). Prefer `import`/`export`.
- Strict TS (`strict: true`), Node16 module resolution.
- Keep generated files untouched:
  - `src/lhar-types.generated.ts`
  - `src/version.generated.ts`

### Formatting and linting

- Use **Biome** for formatting and lint rules.
- Indentation is 2 spaces.
- `any` is discouraged (Biome warns on explicit `any`). Prefer proper typing.

### Tests

- Root tests use **node:test** and live in `test/`.
- Tests are compiled with `tsconfig.test.json` into `dist-test/` and executed via `node --test`.
- Follow existing naming: `*.test.ts`.

### Architecture notes (when extending)

Adding a new tool integration typically touches:

1. Provider detection: `src/core/routing.ts:detectProvider()` if a new request format is needed.
2. Source detection: `src/core/source.ts` (`HEADER_SIGNATURES` / `SOURCE_SIGNATURES`).
3. CLI wiring: `src/cli-utils.ts` for base-URL env vars / per-tool behavior.
4. Tests: add coverage in `test/` for detection/routing/parsing.

MITM mode (for Cloudflare-protected endpoints) uses:

- `mitm_addon.py` (mitmproxy addon) to capture and POST to `/api/ingest` on the analysis server.

### Performance constraints

Startup performance depends on loading `state.jsonl`. Keep `Store.loadState()` and migrations efficient:

- avoid disk I/O inside hot loops
- avoid O(n²) scans across state entries
- prefer append-only writes for new entries (`appendToState()`), full rewrites only for structural changes

(See `CONTRIBUTING.md` for the current performance target and measurement snippet.)

## Data and privacy

Default local persistence (from README):

- `~/.context-lens/data/state.jsonl`
- per-session `.lhar` files in `~/.context-lens/data/`

Be careful when logging or exporting captured content; it may include prompts, tool outputs, or sensitive data depending on `CONTEXT_LENS_PRIVACY`.

## Release / publishing

- Publishing is automated via GitHub Actions using **npm trusted publishing (OIDC)**.
- Update version in `package.json`, push to `main`, create GitHub release (see `CONTRIBUTING.md`).

## Quick reference

- Entry points:
  - CLI: `dist/cli.js`
  - Proxy: `dist/proxy/server.js`
  - Analysis: `dist/analysis/server.js`
- Root scripts: see `package.json`.
