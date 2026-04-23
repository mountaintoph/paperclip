# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is Paperclip

Paperclip is an open-source orchestration platform for teams of AI agents. It coordinates agents (Claude Code, Codex, Cursor, OpenClaw, etc.) toward shared goals with budgets, governance, org charts, and task management.

## Common Commands

```bash
pnpm install          # Install all dependencies
pnpm dev              # Start API + UI in watch mode (idempotent — reuses existing process)
pnpm dev:once         # Start without file watching (auto-applies pending migrations)
pnpm dev:server       # Server only
pnpm dev:ui           # UI only
pnpm dev:list         # Show running dev process for this repo
pnpm dev:stop         # Stop running dev process
pnpm build            # Build all packages
pnpm typecheck        # TypeScript strict check (run before committing)
pnpm test:run         # Run all unit tests (Vitest)
pnpm test:e2e         # Playwright E2E tests
pnpm db:generate      # Generate a new DB migration
pnpm db:migrate       # Apply migrations
```

Running a single test file: `pnpm vitest run path/to/test.ts`

Dev server runs on `http://localhost:3100` (API + UI served from same origin via Vite dev middleware).

### Lockfile Policy

**Do not commit `pnpm-lock.yaml` in pull requests.** CI on master regenerates it automatically. PR CI validates dependency resolution when manifests change.

## Architecture

This is a **pnpm monorepo** with the following workspaces:

```
packages/
  shared/           # Zod-based shared types (@paperclipai/shared)
  db/               # Drizzle ORM schema + migrations (@paperclipai/db)
  adapter-utils/    # Base adapter framework utilities
  adapters/         # Agent runtime adapters (claude-local, codex-local, cursor-local, etc.)
  plugins/          # Plugin SDK + example plugins
  mcp-server/       # Model Context Protocol server package
server/             # Express.js API server (@paperclipai/server)
ui/                 # React 19 frontend (@paperclipai/ui)
cli/                # Commander.js CLI (paperclipai)
```

### Server (`server/src/`)

- `index.ts` — Express app startup, DB connection, auth, WebSocket server
- `config.ts` — All runtime configuration (deployment mode, DB provider, storage, auth)
- `routes/` — One file per API resource (agents, goals, issues, costs, etc.). Each exports a function returning an Express Router.
- `services/` — Business logic services (agents, budgets, costs, cron, approvals, etc.)
- `middleware/` — `validate` (Zod schema validation factory), `actorMiddleware` (auth context), `boardMutationGuard`, `errorHandler`
- `storage/` — File storage provider abstraction (local disk, S3)
- `secrets/` — Secret storage abstraction (local encrypted)
- `realtime/` — WebSocket server on same HTTP server; emits domain events (issue.created, agent.updated, etc.) to connected clients
- `auth/` — Better-auth integration; `local_trusted` mode creates synthetic "local-board" user with instance_admin role

### Frontend (`ui/src/`)

React 19 + Vite + Tailwind CSS + Radix UI. Uses TanStack React Query for data fetching.

- **Routing**: Custom wrapper in `ui/src/lib/router.tsx` re-exports react-router-dom with company-prefix awareness. `useActiveCompanyPrefix()` auto-injects company prefix into Link/NavLink/Navigate.
- **State**: React Query for server state; React Context providers for UI state (CompanyContext, DialogContext, ToastContext, SidebarContext, ThemeContext, etc.). No Redux/Zustand.
- **Query keys**: Centralized factory in `ui/src/lib/queryKeys.ts`.
- **Live updates**: `LiveUpdatesProvider` manages WebSocket connection for realtime events.
- **Vite config**: Path alias `@` → `./src`. Dev proxy: `/api` → `http://localhost:3100`.

### Shared Types (`packages/shared/`)

Zod schemas for all domain entities. Server routes validate request bodies via `validate()` middleware using these schemas. Frontend imports the same schemas. This prevents client-server type drift. Also exports enum/constant arrays (`AGENT_STATUSES`, `ISSUE_PRIORITIES`, etc.), `API` path constants, and URL key helpers (`deriveAgentUrlKey()`, etc.).

### Database

PostgreSQL via Drizzle ORM. Schema lives in `packages/db/`. In development, an **embedded PostgreSQL** instance runs automatically on port 54329 — no external DB needed. Migrations run automatically on `pnpm dev`. Set `DATABASE_URL` env var to use external Postgres instead.

### Adapter System

Each agent runtime is an adapter package in `packages/adapters/`. Adapters implement the interface from `@paperclipai/adapter-utils` (`AdapterRuntime`, `AdapterAgent`). Registered via `packages/adapters/registry.ts`. To add a new agent type, create a new adapter package following existing examples.

### Plugin System

Plugins run in **isolated Worker processes** (not in the main server). SDK in `packages/plugins/sdk/` exports `definePlugin()` + `runWorker()`. Plugin discovery scans `~/.paperclip/plugins/` + node_modules for `paperclip-plugin-*` packages. Host-side services: PluginWorkerManager, PluginEventBus, PluginJobScheduler, PluginToolDispatcher, PluginLifecycleManager. Manifests validated by capability and schema validators.

### MCP Server (`packages/mcp-server/`)

Thin wrapper exposing Paperclip as an MCP server. All calls go through REST API (no direct DB access). Config via env vars: `PAPERCLIP_API_URL`, `PAPERCLIP_API_KEY`, `PAPERCLIP_COMPANY_ID`, `PAPERCLIP_AGENT_ID`.

### Configuration

Runtime config is read from `~/.paperclip/instances/default/config.json`. Key options: deployment mode (`local_trusted` vs `authenticated`), DB provider, storage provider, secrets provider. Use `pnpm paperclipai configure` to edit.

Key env vars: `PAPERCLIP_DEPLOYMENT_MODE`, `PAPERCLIP_STORAGE_PROVIDER`, `PAPERCLIP_SECRETS_PROVIDER`, `HEARTBEAT_SCHEDULER_ENABLED`, `PAPERCLIP_DB_BACKUP_ENABLED`.

### Testing

Tests colocated with source in each workspace:
- Server: `server/src/__tests__/`
- CLI: `cli/src/__tests__/` (includes embedded-postgres test helpers in `helpers/`)
- Adapters: each adapter has its own `vitest.config.ts`

## Key Documentation

More detailed docs are in `doc/`:
- `doc/DEVELOPING.md` — Dev environment setup details
- `doc/SPEC.md` — Data model spec (companies, agents, goals, tasks)
- `doc/DATABASE.md` — Database schema and structure
- `doc/DOCKER.md` — Docker deployment
- `doc/CLI.md` — CLI usage
- `doc/DEPLOYMENT-MODES.md` — `local_trusted` vs `authenticated` mode definitions

## Design System

When working on the UI, use the `/design-guide` skill for Paperclip's design system conventions. The design showcase page is at `/design-guide` in the running app.
