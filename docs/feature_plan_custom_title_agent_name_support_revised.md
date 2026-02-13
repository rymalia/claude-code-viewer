# Plan: Support `custom-title` and `agent-name` JSONL Entry Types

## Context

When a user runs `/rename` in Claude Code, two new JSONL entry types are written to the session log:
- `{"type":"custom-title","customTitle":"...","sessionId":"..."}`
- `{"type":"agent-name","agentName":"...","sessionId":"..."}`

Claude Code Viewer's Zod schema doesn't recognize these types, so they appear as red "Schema Validation Error" cards in the conversation view. This plan adds proper schema support, uses the custom title as the session's display name, and makes it searchable.

The work is split into **two independent plans**, each producing its own PR:
- **Plan 1** fixes schema parse errors and type cascade issues — the "make it stop erroring" work
- **Plan 2** adds feature behavior — displaying and indexing custom titles — the "make it useful" work

Each plan leaves the codebase in a compiling, test-passing state. They can be implemented in separate sessions without risk.

## Design Decisions

- **Multiple renames**: A user can `/rename` multiple times. Always use the **last** `custom-title` entry in the JSONL (most recent rename wins). Iterate the full `lines` array rather than stopping at the first match.
- **`agent-name` scope**: For now, `agent-name` is parsed but not stored or displayed. It appears to be an internal agent identifier. We may revisit displaying it later, but the current scope is: parse it (no more schema errors), hide it from rendering, and ignore it in search/metadata.
- **Cache invalidation**: `SessionMetaService` already caches `SessionMeta` in a `Ref` and invalidates via the existing `invalidateSession()` method triggered by SSE `sessionChanged` events. This existing mechanism is sufficient — no new invalidation logic needed. When a user `/rename`s a live session, the JSONL file changes, SSE fires, cache invalidates, and the next read picks up the new `customTitle`.
- **Type cascade**: After adding the new schemas to `ConversationSchema`, the `Conversation` type (`z.infer<typeof ConversationSchema>`) automatically widens to include the new types. This will cause TypeScript errors in files that do exhaustive type narrowing (e.g., `getConversationKey`'s `satisfies never`) **and** in files that access fields like `.isSidechain` or `.uuid` after incomplete type narrowing (e.g., `useSidechain.ts`, the second `isSidechain` check in `ConversationList.tsx`). TypeScript does not narrow types through non-type-guard function calls — even if `shouldRenderConversation()` filters the new types at runtime, TypeScript still sees them in subsequent code. Plan 1 fixes all of these.

---

## Plan 1: Fix Schema Parse Errors *(PR #1)*

### Approach

This plan is guided by the project's `/fix-schema` command workflow, which defines a proven pattern for this exact problem: a new JSONL type appears, Zod doesn't recognize it, and the type cascade needs fixing. The `/fix-schema` workflow steps are:

1. Locate and update schema
2. Fix type errors
3. Add tests
4. Verify and commit
5. Create PR

We hit all five steps, but **reorder for TDD** — tests are written before implementation in each logical group. The `/fix-schema` command is a prompt template (not a rigid script), so this reordering preserves its intent while matching the project's TDD discipline.

**`/fix-schema` principles** (all apply to Plan 1):
- **Backward compatibility**: Existing JSONL files must continue to parse
- **Type safety**: Follow project rules (no `as` casting)
- **Atomic commits**: All related changes in single commit (schema + UI + tests)
- **Pattern consistency**: Follow existing schema patterns in codebase

**Input data** (the two failing JSONL entries):
```json
{"type":"custom-title","customTitle":"My Custom Name","sessionId":"abc-123"}
{"type":"agent-name","agentName":"claude-code-agent","sessionId":"abc-123"}
```

> **TDD Note**: For each step below, write the relevant test(s) **before** writing the implementation code. The tests serve as the specification — write them, watch them fail, then implement to make them pass.

### Step 1: Schema — Parse the new entry types

**Tests first** (modify `src/server/core/claude-code/functions/parseJsonl.test.ts`):
- `custom-title` entry parses correctly (standalone)
- `agent-name` entry parses correctly (standalone)
- Both types mixed with regular user/assistant/summary entries parse correctly (no `x-error`)

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

### Step 2: Search text extraction — Handle new types

**Tests first** (modify `src/server/core/search/functions/extractSearchableText.test.ts`):
- Returns `customTitle` string for `custom-title` entries
- Returns `null` for `agent-name` entries

**Then implement:**

**Modify** `src/server/core/search/functions/extractSearchableText.ts`
- Add a case: if `conversation.type === "custom-title"`, return `conversation.customTitle`
- Add a case: if `conversation.type === "agent-name"`, return `null` (not useful for search)
- Note: `extractSearchableText` uses a catch-all `return null` (no `satisfies never` exhaustive check), so these cases are not strictly required by TypeScript — but they make the intent explicit and prevent the new types from being silently ignored if the function is later refactored to be exhaustive

### Step 3: Rendering — Fix type cascade in conversation view

**No unit tests** — these are compile-time correctness fixes. Run `pnpm typecheck` to verify.

**Modify** `src/app/projects/[projectId]/sessions/[sessionId]/components/conversationList/ConversationList.tsx`

- `getConversationKey()` (line 36): Add cases for `"custom-title"` and `"agent-name"` so the `satisfies never` exhaustive check passes. Use format: `custom-title_${conversation.sessionId}_${conversation.customTitle}` and `agent-name_${conversation.sessionId}_${conversation.agentName}` (include the value to avoid duplicate keys when a user `/rename`s multiple times)
- `shouldRenderConversation()` (line 283): Return `false` for these types **before** the `isSidechain` check (line 290-294). This is important because `custom-title` and `agent-name` don't have an `isSidechain` field — if the check reaches the isSidechain line, it would fail. Add them alongside the existing `if (conv.type === "progress") return false;` pattern.
- **Second `isSidechain` check** (line 390-395): There is a second `isSidechain` access in the rendering path's layout logic. Even though `shouldRenderConversation()` filters the new types at runtime, TypeScript doesn't narrow through non-type-guard functions, so the new types are still visible at this point. Add `conversation.type !== "custom-title" && conversation.type !== "agent-name"` to the condition:
  ```typescript
  const isSidechain =
    conversation.type !== "summary" &&
    conversation.type !== "file-history-snapshot" &&
    conversation.type !== "queue-operation" &&
    conversation.type !== "progress" &&
    conversation.type !== "custom-title" &&
    conversation.type !== "agent-name" &&
    conversation.isSidechain;
  ```
- Timestamp visibility logic (line 319): Add `"custom-title"` and `"agent-name"` to the list alongside `"summary"`, `"progress"`, `"queue-operation"`, `"file-history-snapshot"` that gets `showTimestamp: false`

**Modify** `src/app/projects/[projectId]/sessions/[sessionId]/hooks/useSidechain.ts`

This file operates on `Conversation[]` and accesses `.isSidechain` and `.uuid` after incomplete type narrowing. Two fixes needed:

- Filter at lines 12-15: Add `"custom-title"` and `"agent-name"` to the type exclusion list so they don't reach the `.isSidechain` access on line 17:
  ```typescript
  .filter(
    (conv) =>
      conv.type !== "summary" &&
      conv.type !== "file-history-snapshot" &&
      conv.type !== "queue-operation" &&
      conv.type !== "progress" &&
      conv.type !== "custom-title" &&
      conv.type !== "agent-name",
  )
  .filter((conv) => conv.isSidechain === true);
  ```
- `isRootSidechain()` guard at lines 96-99: Add `"custom-title"` and `"agent-name"` to the early-return guard so they don't reach the `.uuid` access on line 104:
  ```typescript
  if (
    conversation.type === "summary" ||
    conversation.type === "file-history-snapshot" ||
    conversation.type === "queue-operation" ||
    conversation.type === "custom-title" ||
    conversation.type === "agent-name"
  ) {
    return false;
  }
  ```

### Verification

1. `pnpm typecheck` — must pass (all type cascade errors resolved)
2. `pnpm test` — must pass (**ask user before running** — memory-intensive)
3. `pnpm fix` — lint/format
4. **Manual check**: Open a session that was `/rename`d in Claude Code Viewer — the red "Schema Validation Error" cards should be gone

### Commit & PR

- All Plan 1 changes in a single atomic commit (schema + type cascade + tests)
- Push branch, create draft PR with summary of schema changes
- Suggested commit type: `fix: support custom-title and agent-name JSONL entry types`

### Files (Plan 1)

| File | Action |
|------|--------|
| `src/lib/conversation-schema/entry/CustomTitleEntrySchema.ts` | **Create** |
| `src/lib/conversation-schema/entry/AgentNameEntrySchema.ts` | **Create** |
| `src/lib/conversation-schema/index.ts` | Modify |
| `src/app/.../conversationList/ConversationList.tsx` | Modify |
| `src/app/.../hooks/useSidechain.ts` | Modify |
| `src/server/core/search/functions/extractSearchableText.ts` | Modify |
| `src/server/core/claude-code/functions/parseJsonl.test.ts` | Modify |
| `src/server/core/search/functions/extractSearchableText.test.ts` | Modify |

### After Plan 1

- No more red "Schema Validation Error" cards for `custom-title` and `agent-name` entries
- New entry types are parsed, hidden from conversation view, and ignored in sidechain logic
- Custom title text is returned by `extractSearchableText` (ready for Plan 2's indexing)
- Project compiles cleanly with no type errors
- All tests pass

---

## Plan 2: Display and Index Custom Titles *(PR #2)*

### Approach

This is manual feature development — **not** a `/fix-schema` invocation. The `/fix-schema` command is designed for schema parse error fixes; Plan 2 is additive feature work (metadata extraction, UI display, search indexing).

Plan 1's clean compile guarantees a stable starting point. All new types are already parsed and hidden; this plan makes them *useful*.

> **TDD Note**: For each step below, write the relevant test(s) **before** writing the implementation code. Steps without testable behavior (UI display, search wiring) note this explicitly.

### Step 1: Backend — Add `customTitle` to session metadata

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

### Step 2: Frontend — Display custom title in the UI

**No unit tests** — UI display logic verified by manual inspection.

Four places currently compute session titles. Three share the same pattern:
```typescript
const title = session.meta.firstUserMessage !== null
  ? firstUserMessageToTitle(session.meta.firstUserMessage)
  : session.id;
```

Change these three to prioritize `customTitle`:
```typescript
const title = session.meta.customTitle
  ?? (session.meta.firstUserMessage !== null
    ? firstUserMessageToTitle(session.meta.firstUserMessage)
    : session.id);
```

Files with the shared pattern:
- `src/app/projects/[projectId]/sessions/[sessionId]/components/sessionSidebar/SessionsTab.tsx` (line 112)
- `src/app/components/SessionHistoryPopover.tsx` (line 171)
- `src/app/projects/[projectId]/sessions/[sessionId]/components/SessionPageMain.tsx` (lines 588-593)

**Note — different pattern at `SessionPageMain.tsx` line 256-259**: This location uses different variable names (`sessionData?.session.meta` with optional chaining, and `sessionId ?? ""` as fallback instead of `session.id`). Adapt the template accordingly:
```typescript
const sessionTitle = sessionData?.session.meta.customTitle
  ?? (sessionData?.session.meta.firstUserMessage != null
    ? firstUserMessageToTitle(sessionData.session.meta.firstUserMessage)
    : (sessionId ?? ""));
```

### Step 3: Search — Index custom titles for search

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

### Verification

1. `pnpm typecheck` — must pass
2. `pnpm test` — must pass (**ask user before running** — memory-intensive)
3. `pnpm fix` — lint/format
4. **Manual check**: Open a session that was `/rename`d in Claude Code Viewer:
   - The custom title should appear in the sidebar session list
   - The custom title should appear in the session history popover
   - The custom title should appear in the session header
5. **Search check**: Use the search feature to search for the custom title text — it should match and return the session

### Commit & PR

- All Plan 2 changes in a single atomic commit (metadata + UI + search)
- Push branch, create draft PR with summary of feature changes
- Suggested commit type: `feat: display custom session titles from /rename command`

### Files (Plan 2)

| File | Action |
|------|--------|
| `src/server/core/session/schema.ts` | Modify |
| `src/server/core/session/services/SessionMetaService.ts` | Modify |
| `src/server/core/session/testing/createMockSessionMeta.ts` | Modify |
| `src/server/core/session/services/SessionMetaService.test.ts` | Modify |
| `src/app/.../sessionSidebar/SessionsTab.tsx` | Modify |
| `src/app/components/SessionHistoryPopover.tsx` | Modify |
| `src/app/.../SessionPageMain.tsx` | Modify |
| `src/server/core/search/services/SearchService.ts` | Modify |

### After Plan 2

- Custom titles display in sidebar, popover, and session header
- Custom titles are searchable
- Multiple renames are handled correctly (last one wins)
- Full feature complete

---

## Why This Split Works

1. **Plan 1 maps to `/fix-schema`'s design intent** — it's a schema fix with type cascade cleanup, exactly what the command was built for. Plan 2 is feature development, a fundamentally different kind of work.
2. **Clean boundary** — Plan 1 leaves the project in a compiling, test-passing state; Plan 2 is purely additive. Each produces its own PR.
3. **Context efficiency** — Plan 1 touches 8 files (mostly small edits); Plan 2 touches 8 files (also small edits). Neither risks exhausting the context window.
4. **Independent verifiability** — after Plan 1, open a renamed session and confirm the red error cards are gone. After Plan 2, confirm the custom title displays and is searchable.
5. **Separate PRs** — a reviewer can evaluate "does this fix the schema errors?" independently from "does the custom title display correctly?" The first is a bugfix (`fix:`), the second is a feature (`feat:`).

---

## Test Inventory

All tests to be written (listed here for reference; write each test inline with its implementation step following TDD):

### `parseJsonl.test.ts` *(Plan 1, Step 1)*
1. `custom-title` entry parses correctly (standalone)
2. `agent-name` entry parses correctly (standalone)
3. Both types mixed with regular user/assistant/summary entries parse without `x-error`

### `extractSearchableText.test.ts` *(Plan 1, Step 2)*
4. Returns `customTitle` string for `custom-title` entries
5. Returns `null` for `agent-name` entries

### `SessionMetaService.test.ts` *(Plan 2, Step 1)*
6. `customTitle` is extracted when a `custom-title` entry exists
7. `customTitle` is `null` when no `custom-title` entry exists
8. When multiple `custom-title` entries exist, the last one wins
9. `customTitle` extraction works regardless of entry position (beginning, middle, end of JSONL)
