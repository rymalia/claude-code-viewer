# Feature Plan: Handle Custom Session Metadata (`custom-title`, `agent-name`)

This document outlines the findings and plan for resolving schema validation errors caused by Claude Code's `/rename` command and enabling these fields in the UI and search.

## 1. Findings

### Cause of Schema Validation Errors
- **Incompatible Schema:** The `ConversationSchema` in `src/lib/conversation-schema/index.ts` does not include the `custom-title` and `agent-name` event types.
- **Missing Base Fields:** These new entries (e.g., `{"type":"custom-title","customTitle":"...","sessionId":"..."}`) do not extend `BaseEntrySchema`. They lack required fields like `uuid`, `timestamp`, `cwd`, etc.
- **Parsing Failure:** `src/server/core/claude-code/functions/parseJsonl.ts` uses `ConversationSchema.safeParse`, which fails for these lines. It returns an `ErrorJsonl` object, rendered as a "Schema Validation Error" in the UI.

### Current Metadata & Search Limitations
- **Session Titles:** Currently derived from the first user message in `SessionMetaService.ts`. Custom titles are ignored.
- **Search:** `extractSearchableText.ts` only handles `user` and `assistant` types. Custom titles and agent names are not indexed.

## 2. Implementation Plan

### Step 1: Schema Updates
1.  **Create** `src/lib/conversation-schema/entry/MetadataEntrySchema.ts`:
    - Define `CustomTitleEntrySchema` (fields: `type`, `customTitle`, `sessionId`).
    - Define `AgentNameEntrySchema` (fields: `type`, `agentName`, `sessionId`).
2.  **Update** `src/lib/conversation-schema/index.ts`:
    - Add `MetadataEntrySchema` to the `ConversationSchema` union.
3.  **Update** `src/server/core/session/schema.ts`:
    - Add `customTitle` and `agentName` (optional strings) to `sessionMetaSchema`.

### Step 2: Backend Logic Updates
1.  **Update** `src/server/core/session/services/SessionMetaService.ts`:
    - Modify `getSessionMeta` to scan conversation entries for `custom-title` and `agent-name` and populate `SessionMeta`.
    - **Logic:** Use the **last** occurrence of each type in the file (last-one-wins), as Claude Code allows multiple renames.
2.  **Update** `src/server/core/search/functions/extractSearchableText.ts`:
    - Add logic to return `customTitle` or `agentName` text for indexing.
3.  **Update** `src/server/core/search/services/SearchService.ts`:
    - Update the type guard (line ~115) to allow `custom-title` and `agent-name` types so they are not skipped during indexing.

### Step 3: Frontend UI Updates
1.  **Update** Title Rendering:
    - Priority: `session.meta.customTitle` -> `firstUserMessageToTitle(session.meta.firstUserMessage)` -> "Untitled".
2.  **Update** Agent Name Display:
    - Display `session.meta.agentName` in the session metadata popover (near model name) if it differs from the default.
3.  **Files to Update**:
    - `src/app/projects/[projectId]/sessions/[sessionId]/components/SessionPageMain.tsx`
    - `src/app/projects/[projectId]/sessions/[sessionId]/components/sessionSidebar/SessionsTab.tsx`
    - `src/app/components/SessionHistoryPopover.tsx`
    - `src/app/components/SessionMetadataPopover.tsx` (or equivalent where model info is shown)

## 3. Detailed Specifications

### Metadata Entry Schema
```typescript
export const CustomTitleEntrySchema = z.object({
  type: z.literal("custom-title"),
  customTitle: z.string(),
  sessionId: z.string(),
});

export const AgentNameEntrySchema = z.object({
  type: z.literal("agent-name"),
  agentName: z.string(),
  sessionId: z.string(),
});
```

### Search Extraction Logic
```typescript
if (conversation.type === "custom-title") {
  return conversation.customTitle;
}
if (conversation.type === "agent-name") {
  return conversation.agentName;
}
```

## 4. Verification & Testing

### Success Criteria
1.  **No Schema Errors:** Sessions containing `/rename` commands no longer show red "Schema Validation Error" boxes.
2.  **Custom Titles:** The UI displays the renamed session title in the sidebar, header, and history popover.
3.  **Searchability:** Searching for a custom session title or agent name returns that session in the search results.

### Automated Tests
1.  **Update** `src/server/core/session/services/SessionMetaService.test.ts`: Add a test case with a mock session file containing `custom-title` and `agent-name` entries.
2.  **Update** `src/server/core/search/functions/extractSearchableText.test.ts`: Add test cases for the new metadata types.
3.  **Run** `pnpm test` and `pnpm typecheck` to ensure system integrity.
