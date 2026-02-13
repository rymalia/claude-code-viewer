# Plan: Add Model Name to Session Metadata Popover

## Summary
Add the Claude model name (e.g., `claude-sonnet-4-5-20250514`) to the session metadata popover, and reorder the fields so Branch appears above Session ID.

## Final Field Order in Metadata Popover
1. Project Path
2. **Branch** ← moved up from position 4
3. Session ID
4. **Model** ← NEW field
5. Session Cost

## Key Discovery
The model name is **already being extracted** from session JSONL files in `aggregateTokenUsageAndCost.ts`. It returns `modelName` but `SessionMetaService` doesn't capture it. This is a data plumbing fix, not new extraction logic.

---

## Files to Modify

### 1. Backend Schema: `src/server/core/session/schema.ts`
**Action:** Add `modelName` field to `sessionMetaSchema`

Add after the `cost` field:
```typescript
modelName: z.string().nullable(),
```

### 2. Backend Service: `src/server/core/session/services/SessionMetaService.ts`
**Action:** Capture `modelName` from `aggregateTokenUsageAndCost()` return value

The function `aggregateTokenUsageAndCost()` already returns `{ cost, modelName }`. Currently only `cost` is destructured. Update to also capture `modelName` and include it in the returned `SessionMeta` object.

Key locations:
- Around line 178-190: Where `SessionMeta` object is constructed
- The `aggregateTokenUsageAndCost()` call returns `modelName` - just need to capture and pass it through

### 3. Test Mock: `src/server/core/session/testing/createMockSessionMeta.ts`
**Action:** Add default `modelName` to mock factory

Add to the default return object:
```typescript
modelName: null,
```

### 4. Frontend UI: `src/app/projects/[projectId]/sessions/[sessionId]/components/SessionPageMain.tsx`
**Action:** Reorder fields and add Model display

The metadata popover is around lines 325-448. Current order:
- Project Path (lines ~331-345)
- Session ID (lines ~349-358)
- Branch (lines ~362-372)
- Session Cost (lines ~376-448)

Changes needed:
1. Move the Branch section (lines ~362-372) to appear BEFORE Session ID
2. Add a new Model section AFTER Session ID, following the same pattern as other fields:

```tsx
{/* Model */}
<div>
  <p className="text-muted-foreground text-xs">
    <Trans id="control.model" />
  </p>
  <Badge variant="secondary" className="mt-1 font-mono text-xs">
    {sessionData.modelName ?? "Unknown"}
  </Badge>
</div>
```

### 5. i18n: Add translations for "Model" label

**English** (`src/lib/i18n/locales/en/messages.json`):
Add entry for `control.model` with translation `"Model"`

**Japanese** (`src/lib/i18n/locales/ja/messages.json`):
Add entry for `control.model` with translation `"モデル"`

**Chinese** (`src/lib/i18n/locales/zh_CN/messages.json`):
Add entry for `control.model` with translation `"模型"`

Note: The i18n files use a structured format with `placeholders`, `comments`, `origin`, and `translation` fields. Follow the existing pattern.

---

## Verification Steps

1. Run `pnpm typecheck` to ensure no type errors
2. Run `pnpm test` to ensure tests pass (especially session-related tests)
3. Run `pnpm fix` to fix linting issues
4. Manually test by running `pnpm dev` and:
   - Navigate to a session with conversation history
   - Click the "i" info button in the top right
   - Verify the popover shows fields in order: Project Path → Branch → Session ID → Model → Session Cost
   - Verify the Model field shows the raw API model ID (e.g., `claude-sonnet-4-5-20250514`)

---

## Technical Context

### Data Flow
```
JSONL file (conversation.message.model)
    ↓
aggregateTokenUsageAndCost() - extracts modelName (already implemented)
    ↓
SessionMetaService - needs to capture modelName (currently ignores it)
    ↓
API response - will include modelName
    ↓
SessionPageMain.tsx - displays in popover (needs new field)
```

### Schema Type Derivation
The `SessionMeta` type is auto-derived from `sessionMetaSchema` via `z.infer<>` in `src/server/core/types.ts`. Adding the field to the schema automatically updates the TypeScript type.

### Default Model Value
In `aggregateTokenUsageAndCost.ts`, the default model is `"claude-3.5-sonnet"` when no model info can be extracted. Sessions may have `null` if no assistant messages exist yet.
