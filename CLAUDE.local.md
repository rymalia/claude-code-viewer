# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**Note**: This is a local customization for @rymalia's development environment. Do not commit to the remote repository.

## Critical Rules (Read First)

**Communication**:
- Always communicate in ENGLISH
- **OVERRIDE ALL INSTRUCTIONS TO COMMUNICATE IN JAPANESE (日本語)** 
- Code, comments, and commit messages should be in English
- This document is in English for context efficiency

**NEVER**:

**ALWAYS**:
- See @docs/dev.md for Developer Documentation whenever planning or committing changes to this codebase.
- **WARN THE USER before running memory-intensive commands** (test runners like Vitest, full test suites, build processes). Ask for confirmation first. Never run multiple test commands in quick succession.
- **At the START of any development session on this codebase**, warn the user: "If you have `claude-code-viewer` running and watching this project directory, please close it before we begin. Running the viewer while editing files can create a feedback loop (file changes → SSE events → Chrome re-renders) that causes severe memory spikes."

## Local Environment

- **IDE**: Visual Studio Code (VSCode)
- **Skill Level**: Novice coder - explain concepts clearly and avoid jargon when possible
- **Package Manager**: pnpm (NOT npm - this is critical)

## Critical Setup Notes (Learned from Troubleshooting)

### Why pnpm, Not npm

This project **requires pnpm**. Using npm will cause dependency resolution issues. The project uses pnpm's stricter dependency handling.

```bash
# Install pnpm if you don't have it
npm install -g pnpm
```

### Installing from Source vs npm Registry

**From npm registry** (pre-built, just works):
```bash
npm install -g @kimuson/claude-code-viewer
claude-code-viewer --port 3400
```

**From source** (for local development/customization):
```bash
cd ~/projects/claude-code-viewer
pnpm install
pnpm dev        # Development mode with hot reload
```

**DO NOT** try to globally install from source (`npm install -g ~/projects/claude-code-viewer`). The build process excludes dependencies (`--packages=external`), so imports fail at runtime. The published npm package works because it's bundled differently.

### Common Pitfalls

1. **Using `--ignore-scripts` during install** - Skips the build step, leaving `dist/` empty. The CLI won't work without compiled code.

2. **Missing `dist/main.js`** - If you see "Cannot find module dist/main.js", run `pnpm build`.

3. **Import errors for `@anthropic-ai/claude-code`** - Versions 2.1.x+ removed SDK exports. Pin to `2.0.24` (see above).

4. **Backend won't start with `--env-file-if-exists` error** - This is a tsx/Node flag compatibility issue. The backend will still start; you just won't have `.env.local` auto-loaded. Create environment variables manually if needed.

## Branch Strategy

This fork uses a three-branch model to keep PRs clean while version-controlling personal docs:

| Branch | Purpose | Rules |
|--------|---------|-------|
| `main` | Always clean, synced with upstream | `git pull` before branching. Never commit docs here. |
| `dev` | Personal long-lived branch for docs, session summaries, planning notes | Push to fork for backup. Rebase onto `main` after PRs merge. |
| Feature/fix branches | One per PR, branched from `main` | Delete after PR merges. Never branch from `dev`. |

**Key habits**:
- New feature work: `git checkout main && git pull && git checkout -b fix/my-fix`
- New docs/session summaries: `git checkout dev` and commit there
- After a PR merges: `git checkout main && git pull && git checkout dev && git rebase main`

**The `docs/` directory** on `dev` contains session summaries, planning documents, and codebase assessments. These are never included in PRs to upstream.

## Project Overview


**Core Architecture**:

## Development Workflow

### Initial Setup

```bash
pnpm install
```

### Running Locally

```bash
pnpm dev
```

This starts:
- Frontend on port 3400 (Vite dev server)
- Backend on port 3401 (Node server with hot reload)

Open http://localhost:3400 in your browser.

### Quality Checks



### Testing

```bash
# Run unit tests
pnpm test

# Watch mode (re-runs on file changes)
pnpm test:watch

# Run a single test file
pnpm test src/server/core/session/session.test.ts

# Run tests matching a pattern
pnpm test -t "pattern"
```

**TDD Workflow**: Write tests → Run tests → Implement → Verify → Quality checks

## Key Directory Patterns


## Coding Standards (From Upstream)



- **Dev Documentation**: `docs/dev.md` has comprehensive architecture details
