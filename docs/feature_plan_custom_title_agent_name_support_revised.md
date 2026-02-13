# Revised Plan: Support `custom-title` and `agent-name` JSONL Entry Types

## Context

When a user runs `/rename` in Claude Code, two new JSONL entry types are written:
- `{"type":"custom-title","customTitle":"...","sessionId":"..."}`
- `{"type":"agent-name","agentName":"...","sessionId":"..."}`

Claude Code Viewer's Zod schema doesn't recognize these, so they appear as red "Schema Validation Error" cards. The original plan (`docs/feature_plan_custom_title_agent_name_support.md`) has 5 steps across 16 files. This revised plan splits the work into two phases to manage context window limits and leverage the project's existing `/fix-schema` command.

---

## Phase 1: Fix Schema Parse Errors (Session 1)

**Approach:** Invoke `/fix-schema` with the two failing JSONL entries as input. The command's workflow handles schema creation, type cascade fixes, tests, and verification — which maps exactly to Steps 1–2 of the original plan plus the `extractSearchableText` type fix from Step 5.

**Input to provide `/fix-schema`:**
```json
{"type":"custom-title","customTitle":"My Custom Name","sessionId":"abc-123"}
{"type":"agent-name","agentName":"claude-code-agent","sessionId":"abc-123"}
```

### 1. Schema files (original plan Step 1)
- **Create** `src/lib/conversation-schema/entry/CustomTitleEntrySchema.ts` — follow `SummaryEntrySchema` pattern
- **Create** `src/lib/conversation-schema/entry/AgentNameEntrySchema.ts` — same pattern
- **Modify** `src/lib/conversation-schema/index.ts` — add both to `ConversationSchema` union

### 2. Type cascade fixes (original plan Step 2)
- **Modify** `src/app/.../conversationList/ConversationList.tsx`:
  - `getConversationKey()` (line 36): add cases for both types → `satisfies never` fix
  - `shouldRenderConversation()` (line 283): return `false` for both types **before** the `isSidechain` check (line 290) — they lack `isSidechain` field
  - Second `isSidechain` access (line 390): add both types to the exclusion list
  - Timestamp visibility (line 319): add both types to the `showTimestamp: false` list
- **Modify** `src/app/.../hooks/useSidechain.ts`:
  - Filter at lines 10-16: add both types to exclusion list before `.isSidechain` access
  - `isRootSidechain()` guard at lines 96-99: add both types to early-return before `.uuid` access

### 3. Search type fix (original plan Step 5, type-cascade portion only)
- **Modify** `src/server/core/search/functions/extractSearchableText.ts`:
  - Add case: `custom-title` → return `conversation.customTitle`
  - Add case: `agent-name` → return `null`

### 4. Tests
- **Modify** `src/server/core/claude-code/functions/parseJsonl.test.ts`:
  - `custom-title` entry parses correctly (standalone)
  - `agent-name` entry parses correctly (standalone)
  - Both types mixed with regular entries parse without `x-error`
- **Modify** `src/server/core/search/functions/extractSearchableText.test.ts`:
  - Returns `customTitle` string for `custom-title` entries
  - Returns `null` for `agent-name` entries

### 5. Verification
- `pnpm typecheck` — must pass
- `pnpm test` — must pass (ask user before running)
- `pnpm fix` — lint/format

### Files touched in Phase 1
| File | Action |
|------|--------|
| `src/lib/conversation-schema/entry/CustomTitleEntrySchema.ts` | Create |
| `src/lib/conversation-schema/entry/AgentNameEntrySchema.ts` | Create |
| `src/lib/conversation-schema/index.ts` | Modify |
| `src/app/.../conversationList/ConversationList.tsx` | Modify |
| `src/app/.../hooks/useSidechain.ts` | Modify |
| `src/server/core/search/functions/extractSearchableText.ts` | Modify |
| `src/server/core/claude-code/functions/parseJsonl.test.ts` | Modify |
| `src/server/core/search/functions/extractSearchableText.test.ts` | Modify |

### After Phase 1
- No more red "Schema Validation Error" cards
- New entry types are parsed, hidden from view, and ignored in sidechains
- Project compiles cleanly with no type errors
- All tests pass

---

## Phase 2: Feature Work (Session 2)

Separate session using the original plan's Steps 3–5. Context will be fresh, and Phase 1's clean compile guarantees a stable starting point.

### Backend metadata (original plan Step 3)
- **Modify** `src/server/core/session/schema.ts` — add `customTitle: z.string().nullable()`
- **Modify** `src/server/core/session/services/SessionMetaService.ts` — scan JSONL lines for last `custom-title` entry
- **Modify** `src/server/core/session/testing/createMockSessionMeta.ts` — add `customTitle: null`
- **Modify** `src/server/core/session/services/SessionMetaService.test.ts` — 4 new tests

### Frontend display (original plan Step 4)
- **Modify** `src/app/.../sessionSidebar/SessionsTab.tsx` — prioritize `customTitle` in title logic
- **Modify** `src/app/components/SessionHistoryPopover.tsx` — same pattern
- **Modify** `src/app/.../SessionPageMain.tsx` — two locations with different patterns

### Search indexing (original plan Step 5, feature portion)
- **Modify** `src/server/core/search/services/SearchService.ts` — allow `custom-title` through filter, widen `SearchDocument`/`SearchResult` type unions

### Verification
- `pnpm typecheck` + `pnpm test` + `pnpm fix`

### Files touched in Phase 2
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

### After Phase 2
- Custom titles display in sidebar, popover, and header
- Custom titles are searchable
- Full feature complete

---

## Why this split works

1. **Phase 1 maps to `/fix-schema`'s design intent** — it's a schema fix with type cascade cleanup, exactly what the command was built for
2. **Clean boundary** — Phase 1 leaves the project in a compiling, test-passing state; Phase 2 is purely additive
3. **Context efficiency** — Phase 1 touches 8 files (mostly small edits); Phase 2 touches 8 files (also small edits). Neither phase risks exhausting the context window
4. **Independent verifiability** — after Phase 1, you can open a renamed session and confirm the red error cards are gone; after Phase 2, you can confirm the custom title displays

---

## Design Decisions (carried from original plan)

- **Multiple renames**: Always use the **last** `custom-title` entry in the JSONL (most recent rename wins)
- **`agent-name` scope**: Parse but don't store/display. Parse it so no more schema errors, hide from rendering, ignore in search/metadata
- **Cache invalidation**: Existing `SessionMetaService` cache + SSE `sessionChanged` events are sufficient — no new invalidation logic needed
- **Type cascade**: After adding new types to `ConversationSchema`, TypeScript errors will appear in exhaustive checks and `.isSidechain`/`.uuid` accesses. Phase 1 fixes all of these
