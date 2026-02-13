# Plan: Support `custom-title` and `agent-name` JSONL Entry Types

## Context

When a user runs `/rename` in Claude Code, two new JSONL entries are written to the session log:
- `{"type":"custom-title","customTitle":"...","sessionId":"..."}`
- `{"type":"agent-name","agentName":"...","sessionId":"..."}`

Claude Code Viewer's Zod schema doesn't recognize these types, so they appear as red "Schema Validation Error" cards in the conversation view. This plan adds proper schema support, uses the custom title as the session's display name, and makes it searchable.

## Implementation Steps

### Step 1: Create Zod schemas for the two new entry types

**Create** `src/lib/conversation-schema/entry/CustomTitleEntrySchema.ts`
```typescript
// Follow the SummaryEntrySchema pattern — simple object, no BaseEntrySchema
z.object({ type: z.literal("custom-title"), customTitle: z.string(), sessionId: z.string() })
```

**Create** `src/lib/conversation-schema/entry/AgentNameEntrySchema.ts`
```typescript
z.object({ type: z.literal("agent-name"), agentName: z.string(), sessionId: z.string() })
```

**Modify** `src/lib/conversation-schema/index.ts`
- Import both new schemas
- Add them to the `ConversationSchema` z.union array

This alone fixes the schema validation errors — they'll parse correctly instead of becoming `x-error` entries.

### Step 2: Update ConversationList rendering to hide metadata entries

**Modify** `src/app/projects/[projectId]/sessions/[sessionId]/components/conversationList/ConversationList.tsx`

- `getConversationKey()` (line 36): Add cases for `"custom-title"` and `"agent-name"` so the `satisfies never` exhaustive check passes
- `shouldRenderConversation()` (line 283): Return `false` for these types — they're metadata, not renderable messages
- Timestamp visibility logic (line 319): Add these types to the list that gets `showTimestamp: false`

### Step 3: Add `customTitle` to session metadata

**Modify** `src/server/core/session/schema.ts`
- Add `customTitle: z.string().nullable()` to `sessionMetaSchema`

**Modify** `src/server/core/session/services/SessionMetaService.ts`
- In `getSessionMeta()`, scan the raw `lines` array for a `"custom-title"` entry (simple JSON.parse + type check loop, similar to the existing `firstLine` sessionId extraction pattern at lines 117-133)
- Add the extracted `customTitle` (or `null`) to the returned `SessionMeta` object

**Modify** `src/server/core/session/testing/createMockSessionMeta.ts`
- Add `customTitle: null` to the default mock object

### Step 4: Display custom title in the UI

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
- `src/app/projects/[projectId]/sessions/[sessionId]/components/SessionPageMain.tsx` (lines 256-259 and 588-593)

### Step 5: Make custom titles searchable

**Modify** `src/server/core/search/functions/extractSearchableText.ts`
- Add a case: if `conversation.type === "custom-title"`, return `conversation.customTitle`
- For `"agent-name"`, return `null` (not useful for search)

**Modify** `src/server/core/search/services/SearchService.ts`
- In `buildIndex()` (line 111), expand the type filter to also allow `"custom-title"` entries through
- The `SearchDocument` type and `SearchResult` type use `"user" | "assistant"` — we need to widen this to include `"custom-title"` or map it to an existing type. Simplest approach: treat custom-title documents as `type: "user"` in the search index since they represent user-initiated naming

### Step 6: Write tests (TDD — tests first per project convention)

**Modify** `src/server/core/claude-code/functions/parseJsonl.test.ts`
- Add test: `custom-title` entry parses correctly
- Add test: `agent-name` entry parses correctly

**Modify** `src/server/core/session/services/SessionMetaService.test.ts`
- Add test: `customTitle` is extracted from session content
- Add test: `customTitle` is `null` when no custom-title entry exists

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

## Verification

1. **Tests**: Write tests first, then implement — run `pnpm test` after each step
2. **Type check**: `pnpm typecheck` after all changes
3. **Lint/format**: `pnpm fix` before finalizing
4. **Manual check**: Open a session that was `/rename`d — the red error cards should be gone and the custom title should appear in the sidebar, history popover, and session header
5. **Search check**: Search for the custom title text — it should match
