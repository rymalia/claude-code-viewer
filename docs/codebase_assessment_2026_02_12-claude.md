# Codebase Assessment — Claude Code Viewer

**Date:** 2026-02-12
**Assessed by:** Claude (Opus 4.6)
**Baseline:** Session summaries from 2026-01-20
**Scope:** 63 commits, releases v0.5.6 → v0.6.0-beta.2

---

## Major New Features

### Terminal Integration (biggest addition)
- Full terminal feature implemented with PTY backend
- Switched to `ruspty` as the PTY backend
- Terminal CLI options and project cwd support
- UX polish: collapse button, auto-close, restore output after reopening panel

### File Content Viewer
- New API endpoint to retrieve file contents
- View Write/Edit tool history inline (see what Claude changed)
- "Edited files" tab in the sidebar with relative paths
- Support for viewing multiple edited files in a dialog

### Authentication & Security
- Bearer token API auth support
- Auth required for protected API routes and config endpoint

### Session Delete
- Delete session API endpoint + confirmation dialog
- Delete integrated into both sidebar and session info popover

### Rate Limit Auto-Continue
- Detects rate limits from session logs
- `RateLimitAutoScheduleService` auto-continues sessions when rate limit expires
- UI setting to enable/disable

### UI Renewal (PR #147 — large)
- Full PC UI overhaul + mobile support
- Session list and panel layout defaults adjusted
- Session status shown beside title
- Message timestamps in conversation lists
- Improved chat input styling

## Smaller Features

- **Agent sessions section** added to Explorer tab
- **Git branch** shown without needing an active session, right panel state persisted in URL
- **Task management** in session sidebar + DI layer refactor
- **Claude Code options** (model, prompt, etc.) configurable on send message
- **MCP health status** now visible (not just availability)
- **Command completion** enhancements with improved UI text
- **Markdown export** enhanced with more syntax support + tool result rendering
- **Reload button** + granular Suspense boundaries on Git tab

## Architecture / Internal Changes

- **Removed `claude-code` SDK** — dropped v1 support entirely
- **Route refactor** — separated into multiple route files
- **Switched to `tsgo`** for type checking
- **Lefthook** pre-push checks + lingui-extract hook added
- **i18n skill** added with CI translation checks
- **`CCV_ENV`** replaces `NODE_ENV` for dev mode detection

## Bug Fixes

- SSE connection broken after route refactor (fixed)
- Session title empty after `/clear` command
- "No Branch" loading state improved
- Autocomplete indicator visibility
- Local command output handling in sessions

## Notable Observations

The project has gone through a significant growth phase — it's no longer just a log viewer. The terminal integration, file content viewer, and auth layer make it more of a full **session management tool**. The `claude-code` SDK removal is particularly notable — the project now reads JSONL directly without any SDK dependency, which gives it more control but means schema changes in Claude Code need manual tracking.
