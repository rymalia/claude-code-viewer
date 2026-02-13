# GEMINI.md

## Project Overview

**Claude Code Viewer** is a full-featured web-based client for Claude Code, focused on comprehensive session log analysis and real-time interaction. It reads Claude Code's standard session logs directly from JSONL files with zero data loss, providing a modern web interface for managing projects, sessions, and tasks.

### Core Architecture
- **Frontend**: Vite + TanStack Router + React 19 + TanStack Query + Jotai + Tailwind CSS 4.
- **Backend**: Hono (standalone server) + Effect-TS (business logic and side effects).
- **Data Source**: Direct JSONL reads from `~/.claude/projects/` with strict Zod validation.
- **Real-time**: Server-Sent Events (SSE) for live updates to logs and process states.
- **Type Safety**: End-to-end type safety using Hono RPC and strict TypeScript configurations.

---

## Building and Running

### Development
```bash
# Start both frontend and backend development servers
pnpm dev
```
- Frontend: `http://localhost:3400` (proxies `/api` to backend)
- Backend: `http://localhost:3401`

### Production
```bash
# Build the project (compiles i18n, builds frontend and backend)
pnpm build

# Start the production server
pnpm start
```
- Output is generated in the `dist/` directory.
- `dist/main.js` is the entry point for the server and CLI.

### Quality Assurance
```bash
# Run unit tests (Vitest)
pnpm test

# Run type checking (mandatory before commits)
pnpm typecheck

# Check linting and formatting (Biome)
pnpm lint

# Auto-fix linting and formatting
pnpm fix
```

---

## Development Conventions

### General Rules
- **Language**: All code, comments, and commit messages must be in **English**.
- **Commit Messages**: Use [Conventional Commits](https://www.conventionalcommits.org/) (`type: description`). Messages are used in release notes.
- **TDD**: Write tests first, then implement. Verify with `pnpm test`.

### Backend (Effect-TS)
- **Side Effects**: Use Effect-TS for all I/O, async code, and stateful logic.
- **No Node.js Built-ins**: Use `FileSystem.FileSystem`, `Path.Path`, and `Command.string` from Effect-TS instead of `node:fs`, `node:path`, etc.
- **Pure Functions**: Extract logic into pure, testable functions whenever possible.
- **Type Safety**: **Strictly prohibit `as` type casting**. Use type guards, assertion functions, or Zod schemas instead.
- **API Definition**: Define routes using Hono RPC in `src/server/hono/routes/`.

### Frontend
- **API Access**: Use Hono RPC + TanStack Query only. Raw `fetch` is prohibited.
- **State Management**: Use Jotai for global UI state and TanStack Query for server state.
- **Components**: Follow the established pattern using Radix UI and Tailwind CSS 4.

### Quality Standards
- **Linter/Formatter**: Biome (ensure `pnpm fix` is run before committing).
- **TypeScript**: Adhere to `@tsconfig/strictest`.
- **Paths**: Use `@/*` aliases for `src/*`.

---

## Key Directory Patterns
- `src/server/core/`: Domain business logic implemented with Effect-TS.
- `src/server/hono/routes/`: Hono API route definitions.
- `src/lib/conversation-schema/`: Zod schemas for validating Claude Code logs.
- `src/routes/`: TanStack Router file-based routing.
- `src/testing/layers/`: Reusable Effect-TS test layers.
- `mock-global-claude-dir/`: E2E test mocks and reference data structures.
