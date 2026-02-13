# Plan: Support `custom-title` and `agent-name` JSONL Entry Types

## Context

When a user runs `/rename` in Claude Code, two new JSONL entries are written to the session log:
- `{"type":"custom-title","customTitle":"...","sessionId":"..."}`
- `{"type":"agent-name","agentName":"...","sessionId":"..."}`

Claude Code Viewer's Zod schema doesn't recognize these types, so they appear as red "Schema Validation Error" cards in the conversation view. This plan adds proper schema support, uses the custom title as the session's display name, and makes it searchable.

### Design Decisions

- **Multiple renames**: A user can `/rename` multiple times. Always use the **last** `custom-title` entry in the JSONL (most recent rename wins). Iterate the full `lines` array rather than stopping at the first match.
- **`agent-name` scope**: For now, `agent-name` is parsed but not stored or displayed. It appears to be an internal agent identifier. We may revisit displaying it later, but the current scope is: parse it (no more schema errors), hide it from rendering, and ignore it in search/metadata.
- **Cache invalidation**: `SessionMetaService` already caches `SessionMeta` in a `Ref` and invalidates via the existing `invalidateSession()` method triggered by SSE `sessionChanged` events. This existing mechanism is sufficient — no new invalidation logic needed. When a user `/rename`s a live session, the JSONL file changes, SSE fires, cache invalidates, and the next read picks up the new `customTitle`.
- **Type cascade**: After Step 1, the `Conversation` type (`z.infer<typeof ConversationSchema>`) automatically widens to include the new types. This will cause TypeScript errors in files that do exhaustive type narrowing (e.g., `getConversationKey`'s `satisfies never`, `extractSearchableText`'s implicit `return null`). Expect these errors until Steps 2-5 are complete.

## Implementation Steps

> **TDD Note**: This project follows TDD. For each step, write the relevant test(s) **before** writing the implementation code. The test section at the end lists all tests for reference, but they should be written inline with each step.

### Step 1: Schema — Parse the new entry types

**Tests first** (modify `src/server/core/claude-code/functions/parseJsonl.test.ts`):
- `custom-title` entry parses correctly (standalone)
- `agent-name` entry parses correctly (standalone)
- Both types mixed with regular user/assistant entries parse correctly (no `x-error`)

**Then implement:**

**Create** `src/lib/conversation-schema/entry/CustomTitleEntrySchema.ts`
```typescript
// Follow the SummaryEntrySchema pattern — simple object, no BaseEntrySchema
import { z } from "zod";

export const CustomTitleEntrySchema = z.object({
  type: z.literal("custom-title"),
  customTitle: z.string(),
  sessionId: z.string(),
});

export type CustomTitleEntry = z.infer<typeof CustomTitleEntrySchema>;
```

**Create** `src/lib/conversation-schema/entry/AgentNameEntrySchema.ts`
```typescript
import { z } from "zod";

export const AgentNameEntrySchema = z.object({
  type: z.literal("agent-name"),
  agentName: z.string(),
  sessionId: z.string(),
});

export type AgentNameEntry = z.infer<typeof AgentNameEntrySchema>;
```

**Modify** `src/lib/conversation-schema/index.ts`
- Import both new schemas
- Add them to the `ConversationSchema` z.union array
- Export the new types if needed downstream

This alone fixes the schema validation errors — they'll parse correctly instead of becoming `x-error` entries.

### Step 2: Rendering — Hide metadata entries from conversation view

**Modify** `src/app/projects/[projectId]/sessions/[sessionId]/components/conversationList/ConversationList.tsx`

- `getConversationKey()` (line 36): Add cases for `"custom-title"` and `"agent-name"` so the `satisfies never` exhaustive check passes. Use format: `custom-title_${conversation.sessionId}` and `agent-name_${conversation.sessionId}`
- `shouldRenderConversation()` (line 283): Return `false` for these types **before** the `isSidechain` check (line 290-294). This is important because `custom-title` and `agent-name` don't have an `isSidechain` field — if the check reaches the isSidechain line, it would fail. Add them alongside the existing `if (conv.type === "progress") return false;` pattern.
- Timestamp visibility logic (line 319): Add `"custom-title"` and `"agent-name"` to the list alongside `"summary"`, `"progress"`, `"queue-operation"`, `"file-history-snapshot"` that gets `showTimestamp: false`

### Step 3: Backend — Add `customTitle` to session metadata

**Tests first** (modify `src/server/core/session/services/SessionMetaService.test.ts`):
- `customTitle` is extracted when a `custom-title` entry exists in session content
- `customTitle` is `null` when no `custom-title` entry exists
- When multiple `custom-title` entries exist, the **last** one is used (most recent rename wins)
- `customTitle` is extracted correctly regardless of position in the JSONL (beginning, middle, end)

**Then implement:**

**Modify** `src/server/core/session/schema.ts`
- Add `customTitle: z.string().nullable()` to `sessionMetaSchema`

**Modify** `src/server/core/session/services/SessionMetaService.ts`
- In `getSessionMeta()`, scan the raw `lines` array for `"custom-title"` entries using a JSON.parse + type check loop (follow the existing pattern at lines 117-133 where `sessionId` is extracted from the first line). **Iterate all lines and keep the last match** — do not break on first find.
- Add the extracted `customTitle` (or `null`) to the returned `SessionMeta` object at line 163

**Modify** `src/server/core/session/testing/createMockSessionMeta.ts`
- Add `customTitle: null` to the default mock object

### Step 4: Frontend — Display custom title in the UI

Three places currently compute session titles with the same pattern:
```typescript
const title = session.meta.firstUserMessage !== null
  ? firstUserMessageToTitle(session.meta.firstUserMessage)
  : session.id;
```

Change all three to prioritize `customTitle`:
```typescript
const title = session.meta.customTitle
  ?? (session.meta.firstUserMessage !== null
    ? firstUserMessageToTitle(session.meta.firstUserMessage)
    : session.id);
```

Files to modify:
- `src/app/projects/[projectId]/sessions/[sessionId]/components/sessionSidebar/SessionsTab.tsx` (line 112)
- `src/app/components/SessionHistoryPopover.tsx` (line 171)
- `src/app/projects/[projectId]/sessions/[sessionId]/components/SessionPageMain.tsx` (lines 256-259 and 588-593 — two separate title calculations in this file)

### Step 5: Search — Make custom titles searchable

**Modify** `src/server/core/search/functions/extractSearchableText.ts`
- Add a case: if `conversation.type === "custom-title"`, return `conversation.customTitle`
- Add a case: if `conversation.type === "agent-name"`, return `null` (not useful for search)
- These cases must be added before the final `return null` to satisfy TypeScript exhaustiveness

**Modify** `src/server/core/search/services/SearchService.ts`
- In `buildIndex()` (line 111), expand the type filter to also allow `"custom-title"` entries through:
  ```typescript
  if (
    conversation.type !== "user" &&
    conversation.type !== "assistant" &&
    conversation.type !== "custom-title"
  ) {
    continue;
  }
  ```
- **Type handling**: The `SearchDocument` and `SearchResult` types currently use `type: "user" | "assistant"`. Widen this union to `"user" | "assistant" | "custom-title"` in both type definitions (lines 17 and 29). This avoids `as` casting (which is prohibited in this codebase) and keeps the types honest.
- The `timestamp` extraction (line 139-140) uses `"timestamp" in conversation` — since `custom-title` entries don't have a `timestamp` field, this will correctly default to `""`.

## Files Summary

| File | Action |
|------|--------|
| `src/lib/conversation-schema/entry/CustomTitleEntrySchema.ts` | **Create** |
| `src/lib/conversation-schema/entry/AgentNameEntrySchema.ts` | **Create** |
| `src/lib/conversation-schema/index.ts` | Modify |
| `src/app/.../conversationList/ConversationList.tsx` | Modify |
| `src/server/core/session/schema.ts` | Modify |
| `src/server/core/session/services/SessionMetaService.ts` | Modify |
| `src/server/core/session/testing/createMockSessionMeta.ts` | Modify |
| `src/app/.../sessionSidebar/SessionsTab.tsx` | Modify |
| `src/app/components/SessionHistoryPopover.tsx` | Modify |
| `src/app/.../SessionPageMain.tsx` | Modify |
| `src/server/core/search/functions/extractSearchableText.ts` | Modify |
| `src/server/core/search/services/SearchService.ts` | Modify |
| `src/server/core/claude-code/functions/parseJsonl.test.ts` | Modify |
| `src/server/core/session/services/SessionMetaService.test.ts` | Modify |

## Test Inventory

All tests to be written (placed here for reference; write each test inline with its implementation step):

### `parseJsonl.test.ts`
1. `custom-title` entry parses correctly (standalone)
2. `agent-name` entry parses correctly (standalone)
3. Both types mixed with regular user/assistant/summary entries parse without `x-error`

### `SessionMetaService.test.ts`
4. `customTitle` is extracted when a `custom-title` entry exists
5. `customTitle` is `null` when no `custom-title` entry exists
6. When multiple `custom-title` entries exist, the last one wins
7. `customTitle` extraction works regardless of entry position (beginning, middle, end of JSONL)

### `extractSearchableText` (if tests exist for this module, otherwise add to search test file)
8. Returns `customTitle` string for `custom-title` entries
9. Returns `null` for `agent-name` entries

## Verification

1. **Tests**: Run `pnpm test` after each step — tests should pass incrementally
2. **Type check**: Run `pnpm typecheck` after all changes — expect transient type errors between steps (this is fine; all should resolve by end)
3. **Lint/format**: Run `pnpm fix` before finalizing
4. **Manual check**: Open a session that was `/rename`d in Claude Code Viewer:
   - The red "Schema Validation Error" cards should be gone
   - The custom title should appear in the sidebar session list
   - The custom title should appear in the session history popover
   - The custom title should appear in the session header
5. **Search check**: Use the search feature to search for the custom title text — it should match and return the session
