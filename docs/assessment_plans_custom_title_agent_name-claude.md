# Plan Comparison: `custom-title` and `agent-name` Support

Comparative analysis of two proposed development plans for handling `/rename` command JSONL entries.

- **Plan A**: `docs/feature_plan_handle_custom_session_metadata.md`
- **Plan B**: `docs/feature_plan_custom_title_agent_name_support.md`

## Where They Agree (Core Approach)

Both plans correctly identify the same root cause and propose the same fundamental fix:

1. **Root cause**: `ConversationSchema` in `index.ts` is a `z.union` of 7 types. `custom-title` and `agent-name` aren't in that union, so `safeParse` fails and produces `ErrorJsonl` red cards.
2. **Schema design**: Both propose simple flat Zod objects (`type` + payload + `sessionId`) that deliberately do NOT extend `BaseEntrySchema` — matching the existing `SummaryEntrySchema` pattern.
3. **Last-one-wins**: Both agree that multiple `/rename` commands means scanning all lines and keeping the last `custom-title`.
4. **Custom title in metadata**: Both add `customTitle` to `sessionMetaSchema` and `SessionMetaService`.
5. **Search indexing**: Both add `custom-title` to `extractSearchableText`.

This convergence is a good sign — it means the core design is well-reasoned and both developers independently arrived at the right structural approach.

## Where They Diverge (And Who's Right)

### 1. Type Cascade Awareness — Plan B's biggest advantage

When you add a new variant to a Zod `z.union`, the inferred TypeScript type (`z.infer<typeof ConversationSchema>`) widens automatically. Every downstream file that does exhaustive type narrowing (like `satisfies never`) or accesses fields that only exist on *some* variants (like `.isSidechain`, `.uuid`) will break at compile time.

**Plan A** completely ignores this. It never mentions `getConversationKey`, `shouldRenderConversation`, `useSidechain.ts`, or the `satisfies never` exhaustive check.

**Plan B** identifies every single downstream breakage, verified against the actual source:

| Location | Issue | Plan B Correct? |
|----------|-------|-----------------|
| `getConversationKey` (line 65) | `satisfies never` will fail | Yes |
| `shouldRenderConversation` (line 290-294) | `.isSidechain` access on types that lack it | Yes |
| Second `isSidechain` check (line 390-395) | Same `.isSidechain` access in render path | Yes |
| `useSidechain.ts` lines 12-17 | `.isSidechain` access after incomplete filter | Yes |
| `useSidechain.ts` lines 96-99 | `.uuid` access after incomplete guard | Yes |
| Timestamp visibility (line 319) | New types need `showTimestamp: false` | Yes |

**Verdict**: If you followed Plan A as written, `pnpm typecheck` would fail with multiple errors after Step 1. You'd need to discover and fix them on your own. Plan B walks you through every one.

### 2. `agent-name` Scope — Plan B makes a better design decision

**Plan A** treats `agent-name` as a first-class feature: store it in `SessionMeta`, display it in `SessionMetadataPopover`, index it for search.

**Plan B** deliberately scopes it out: parse it (so no more red error cards), but don't store, display, or search it. The rationale: `agent-name` appears to be an internal agent identifier (in the observed example, `agentName` was the same string as `customTitle`). Plan B proposes revisiting display later.

**Verdict**: Plan B's approach is more prudent. You solve the immediate bug (schema errors) without over-building for a field whose semantics aren't fully clear yet.

### 3. File Organization

**Plan A**: One file `MetadataEntrySchema.ts` for both schemas.
**Plan B**: Two files `CustomTitleEntrySchema.ts` and `AgentNameEntrySchema.ts`.

Every existing entry type has its own file (`SummaryEntrySchema.ts`, `ProgressEntrySchema.ts`, etc.). **Plan B matches the established convention.**

### 4. Search Type Safety

**Plan A** says to update `SearchService.ts` line ~115 but doesn't mention the type definitions at lines 17 and 29.

**Plan B** correctly identifies that `SearchDocument` (`type: "user" | "assistant"`) and `SearchResult` (same) need their unions widened to include `"custom-title"`. Without this, you'd need an `as` cast — which is strictly prohibited in this codebase.

### 5. Frontend Specificity

**Plan A**: Vaguely says "Priority: `customTitle` -> `firstUserMessageToTitle()` -> 'Untitled'" and lists 4 files.

**Plan B**: Provides exact code snippets for each of the 4 locations, and calls out that `SessionPageMain.tsx` line 256-259 uses different variable names requiring a slightly adapted template.

### 6. Testing

**Plan A**: 3 bullets pointing at test files.
**Plan B**: 9 specific test cases with a TDD mandate to write them before implementation, organized by file.

### 7. Items Missing from Plan A

| Item | Plan A | Plan B |
|------|--------|--------|
| `getConversationKey` update | Missing | Covered |
| `shouldRenderConversation` update | Missing | Covered |
| `useSidechain.ts` updates (2 locations) | Missing | Covered |
| Second `isSidechain` render check | Missing | Covered |
| Timestamp visibility logic | Missing | Covered |
| `SearchDocument`/`SearchResult` type widening | Missing | Covered |
| `createMockSessionMeta.ts` update | Missing | Covered |
| Cache invalidation analysis | Missing | Covered (existing SSE is sufficient) |
| Transient type error warning | Missing | Covered |

## Will Each Plan Solve the Issue?

**Plan A**: Partially. Step 1 (schema) would fix the red error cards. But Steps 2-3 are incomplete — following them as-is would leave `pnpm typecheck` broken due to unhandled type cascade issues in `ConversationList.tsx`, `useSidechain.ts`, and `SearchService.ts`. A skilled developer could fill in the gaps, but the plan doesn't tell you what those gaps are.

**Plan B**: Yes, fully. Every affected file is identified with specific line-level changes. The 5-step sequence accounts for intermediate type errors and provides a clear path to a green `pnpm typecheck` at the end.

## Summary Table

| Dimension | Plan A | Plan B |
|-----------|--------|--------|
| Root cause analysis | Correct | Correct |
| Schema design | Correct | Correct |
| Type cascade handling | **Missing** | **Thorough** |
| Rendering logic updates | **Missing** | **Complete** |
| `agent-name` scoping | Over-builds | Prudent |
| Frontend specificity | Vague | Exact code snippets |
| Search type safety | Incomplete | Complete |
| Test cases | 3 bullets | 9 specific cases |
| File convention | 1 combined file | 2 files (matches codebase) |
| Implementability | Needs gap-filling | Ready to follow step-by-step |

## Recommendation

**Follow Plan B.** Both plans share a well-designed core (which validates the approach), but Plan B is a complete implementation blueprint that accounts for every downstream consequence of the schema change. Plan A is a reasonable high-level outline that a senior developer could fill in; Plan B traces the ripple effects through every affected file in the codebase.
