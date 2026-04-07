# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What Is Paperclip

Paperclip is an open-source control plane for AI-agent companies. It orchestrates teams of AI agents with org charts, budgeting, governance, cost tracking, and multi-agent coordination. The current implementation target is V1 defined in `doc/SPEC-implementation.md`.

## Commands

```bash
# Install dependencies
pnpm install

# Development (API + UI at http://localhost:3100, auto-restart on changes)
pnpm dev

# Server only / UI only
pnpm dev:server
pnpm dev:ui

# Build all packages
pnpm build

# Typecheck all packages
pnpm -r typecheck

# Tests
pnpm test              # Vitest watch mode
pnpm test:run          # Single run
pnpm test:e2e          # Playwright E2E (headless)
pnpm test:e2e:headed   # Playwright E2E (headed)

# Database
pnpm db:generate       # Generate Drizzle migration after schema changes
pnpm db:migrate        # Apply pending migrations

# Reset local dev DB (uses embedded PGlite when DATABASE_URL unset)
rm -rf data/pglite && pnpm dev

# Verification before hand-off (must all pass)
pnpm -r typecheck && pnpm test:run && pnpm build
```

## Architecture

**Monorepo** using pnpm workspaces. Node.js 20+, TypeScript (ES2023, NodeNext, strict), ES modules throughout.

### Packages

| Package | Path | Purpose |
|---------|------|---------|
| **server** | `server/` | Express 5 REST API, orchestration services, WebSocket realtime |
| **ui** | `ui/` | React 19 + Vite + Tailwind CSS 4 frontend |
| **cli** | `cli/` | `paperclipai` CLI tool (esbuild single-file output) |
| **db** | `packages/db/` | Drizzle ORM schema, migrations, DB client factories |
| **shared** | `packages/shared/` | Types, Zod validators, API path constants |
| **adapter-utils** | `packages/adapter-utils/` | Shared agent adapter utilities |
| **adapters** | `packages/adapters/` | 7 agent adapters (claude, codex, cursor, opencode, pi, gemini, openclaw) |
| **mcp-server** | `packages/mcp-server/` | MCP server exposing Paperclip to AI clients |
| **plugins/sdk** | `packages/plugins/sdk/` | Plugin system public API (v1.0.0) |

### Key Directories

- `server/src/routes/` — 28 Express route files (base path: `/api`)
- `server/src/services/` — 74 business logic service files
- `ui/src/pages/` — 45 page component directories
- `ui/src/components/` — 106 reusable UI component directories
- `doc/` — Product specs, operational docs, plan docs

### Data Flow

All domain entities are **company-scoped**. The server enforces company boundaries on every route. API authentication uses better-auth sessions for board access and bearer API keys (hashed at rest) for agent access.

Embedded PostgreSQL (PGlite) auto-creates at `data/pglite` in dev when `DATABASE_URL` is unset. External PostgreSQL is used in production.

## Control-Plane Invariants (Non-Negotiable)

1. **Single-assignee task model** — one agent per issue at a time
2. **Atomic issue checkout semantics** — no double-work
3. **Approval gates** — governance enforcement before action
4. **Budget hard-stops** — auto-pause when quota exhausted
5. **Activity logging** — immutable audit trail for all mutations

## Contract Synchronization

When changing schema or API behavior, update **all** impacted layers:
1. `packages/db` — schema and exports
2. `packages/shared` — types, constants, validators
3. `server` — routes and services
4. `ui` — API clients and pages

## Database Change Workflow

1. Edit `packages/db/src/schema/*.ts`
2. Ensure new tables are exported from `packages/db/src/schema/index.ts`
3. `pnpm db:generate` (compiles `packages/db` first, reads from `dist/schema/*.js`)
4. `pnpm -r typecheck` to validate

## Adding API Endpoints

- Apply company access checks
- Enforce actor permissions (board vs agent)
- Write activity log entries for mutations
- Return consistent HTTP errors (400/401/403/404/409/422/500)

## Test Projects (vitest.config.ts)

Six test projects: `packages/db`, `packages/adapters/codex-local`, `packages/adapters/opencode-local`, `server`, `ui`, `cli`.

## PR Requirements

PRs **must** use `.github/PULL_REQUEST_TEMPLATE.md`. Required sections: Thinking Path, What Changed, Verification, Risks, Model Used, Checklist.

## Key Docs

- `doc/GOAL.md` — Project vision
- `doc/PRODUCT.md` — Product specs
- `doc/SPEC-implementation.md` — V1 build contract (normative)
- `doc/DEVELOPING.md` — Full development guide
- `doc/DATABASE.md` — Database architecture
- `AGENTS.md` — Contributor guidance (human and AI)
