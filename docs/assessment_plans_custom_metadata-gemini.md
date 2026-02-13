# Assessment: Support for `custom-title` and `agent-name` Plans

This document provides a comparative analysis and assessment of two proposed plans for handling Claude Code's custom metadata entries in the Claude Code Viewer.

## Comparison and Contrast

| Feature | Plan 1 (Custom Title Support) | Plan 2 (Handle Custom Metadata) |
| :--- | :--- | :--- |
| **Strategy** | **Granular & TDD-focused.** Highly specific about codebase side effects. | **Architectural & High-level.** Focuses on the logical flow of data. |
| **TypeScript** | Deeply considers **exhaustive type narrowing** and the "type cascade" effect. | Mentions `typecheck` but doesn't detail the specific breaks in the UI logic. |
| **`agent-name`** | Parses it to avoid errors, but **hides/ignores** it in the UI for now. | Proposes **displaying it** in the metadata popover. |
| **Rendering** | Explicitly fixes potential crashes in `ConversationList` (missing `isSidechain`). | Focuses on title priority logic. |
| **Search** | Widens search types to include `custom-title`. | Adds indexing logic for both types. |

## Assessment

### 1. Design Consistency
The similarity between the two plans is a strong indicator of a well-designed system. Both correctly identify the need to:
- Update the Zod schema to prevent validation errors.
- Modify `SessionMetaService` to scan for the latest title (last-one-wins).
- Update frontend title priority logic.

### 2. Implementation Depth
**Plan 1 is significantly more robust.** It anticipates a critical runtime risk: the existing frontend assumes most conversation entries possess an `isSidechain` property. Because `custom-title` and `agent-name` entries lack this, Plan 1 provides the specific defensive logic needed in `ConversationList.tsx` and `useSidechain.ts` to prevent crashes and TypeScript errors. Plan 2 identifies the root cause but would likely lead to a "broken build" during implementation.

### 3. Problem Resolution
- **Plan 1** provides a comprehensive solution that maintains codebase stability and strict type safety.
- **Plan 2** identifies the fix but misses the necessary UI defensive measures, which would cause runtime issues.

## Insights and Recommendations

- **The "Type Cascade" Insight:** Plan 1's most valuable contribution is anticipating the effects of adding to the `ConversationSchema` union. In a codebase using exhaustive matching (`satisfies never`), this is a breaking change that requires manual updates in several files.
- **Handling `agent-name`:** While Plan 1 chooses to ignore it for now, Plan 2's suggestion to display it in the metadata popover is a useful feature enhancement.
- **Execution Strategy:** Proceed with **Plan 1** as the primary execution guide. It is more grounded in the specific realities of this codebase and will result in a smoother development cycle with fewer unexpected errors.
