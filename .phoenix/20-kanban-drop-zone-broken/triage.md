# Triage Report

## Summary

Kanban card drops only register near the top of a target column when the column is long enough to need scrolling. The `@hello-pangea/dnd` library requires `droppableProvided.placeholder` to be rendered inside the same DOM node that receives `droppableProvided.innerRef`, so that the placeholder reserves space for the dragged card and the library's hit-detection bounding box matches the visible column. In the current fork, `innerRef` is attached to `StyledColumnCardsContainer` (inside `RecordBoardColumnCardsContainer`), but the placeholder is rendered in the sibling `StyledColumn` wrapper inside `RecordBoardColumn`. When a card leaves its origin column, the `innerRef` container shrinks by the card's height, and hit-detection on the target column uses a smaller bounding box than the visible column — exactly the reporter's symptom (drops register only near the top).

A `git log` on the affected files pinpoints a single recent perf commit on `ShankHarinath/twenty` (Nov 11, 2025, "Improved Kanban performance") that simultaneously (a) introduced `DragAndDropLibraryLegacyReRenderBreaker` as a `React.memo` wrapper around `RecordBoardColumnCardsContainer` and (b) moved `{droppableProvided.placeholder}` out of the cards container and up into `StyledColumn`. A previous fix (Oct 2024, `fix: droppable-placeholder`) had specifically placed the placeholder inside `StyledColumnCardsContainer`; the perf refactor undid that, presumably to keep placeholder changes from invalidating the memo boundary and re-rendering the card list during drag.

## Evidence

### Phase A — Code map

Indexed repos consulted: `twenty` (fork at `/Users/shashank/Canonix/phoenix-test/twenty`, pre-fix SHA `31baf52528457c69e44984fe9915d058d3f3384f`). No other repos in scope — this is a frontend-only bug isolated to `packages/twenty-front/src/modules/object-record/record-board`.

Key files inspected (absolute paths):

- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-front/src/modules/object-record/record-board/record-board-column/components/RecordBoardColumn.tsx`
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-front/src/modules/object-record/record-board/record-board-column/components/RecordBoardColumnCardsContainer.tsx`
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-front/src/modules/object-record/record-board/components/RecordBoard.tsx`
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-front/src/modules/object-record/record-board/components/RecordBoardDragDropContext.tsx`
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-front/src/modules/ui/drag-and-drop/components/DragAndDropReRenderBreaker.tsx`

Observations confirmed by reading the current source:

- `RecordBoardColumn.tsx` (L66–80): the `Droppable` render-prop returns `<StyledColumn>` containing `<DragAndDropLibraryLegacyReRenderBreaker memoizationId={recordBoardColumnId}><RecordBoardColumnCardsContainer ... /></DragAndDropLibraryLegacyReRenderBreaker>` followed by `{droppableProvided.placeholder}` as a sibling. Placeholder is OUTSIDE the `innerRef` element.
- `RecordBoardColumnCardsContainer.tsx` (L47–51): `<StyledColumnCardsContainer ref={droppableProvided?.innerRef} {...droppableProvided?.droppableProps}>`. `innerRef` is here; placeholder is not.
- `DragAndDropReRenderBreaker.tsx`: a `React.memo` that only re-renders when `memoizationId` changes, specifically documented with the comment "THIS IS REQUIRED BY THE DND LYBRARY ONLY" — confirms this memo boundary is what made the previous author move the placeholder outside it.
- `RecordBoard.tsx` wraps the whole board in `ScrollWrapper`. `RecordBoardDragDropContext.tsx` uses a plain `DragDropContext` with `onDragStart`/`onDragEnd` only; no `ignoreContainerClipping` is set on any `Droppable` (grep returned zero matches in `record-board`).

GitNexus `impact("RecordBoardColumnCardsContainer", upstream)` returns LOW risk, 0 direct callers reachable via CALLS edges — the component is only JSX-mounted from `RecordBoardColumn.tsx`, so any fix here is contained to these two files.

### Phase B — Runtime logs

Skipped intentionally. This is a pure frontend UX bug — drops silently fail with no exception or log line. The issue body itself lists "Error signatures: none". `kind-canonix` has no frontend pod whose logs could confirm this behavior; reproduction is a browser-level interaction, not a server event.

### Phase C — Recent commits on affected files

```
git log --oneline -- RecordBoardColumn.tsx RecordBoardColumnCardsContainer.tsx
8ef99671e2 Fix board drag and drop issues
ef499b6d47 Re-enable disabled lint rules and right-size CI runners
9d57bc39e5 Migrate from ESLint to OxLint
7a2e397ad1 Complete linaria migration
...
703d9cb08d Improved Kanban performance      <-- SUSPECT
...
d3e503c564 fix: droppable-placeholder       <-- PRIOR FIX (undone by the commit above)
```

`git show 703d9cb08d` on the two files shows, in a single commit:

1. A new import of `DragAndDropLibraryLegacyReRenderBreaker`.
2. The card container wrapped in that memo: `<DragAndDropLibraryLegacyReRenderBreaker memoizationId={recordBoardColumnId}><RecordBoardColumnCardsContainer ... /></DragAndDropLibraryLegacyReRenderBreaker>`.
3. The line `{droppableProvided.placeholder}` added to `RecordBoardColumn` right after the memo wrapper (i.e., outside the cards container).
4. In `RecordBoardColumnCardsContainer.tsx`, the previous `{droppableProvided?.placeholder}` line (added by the Oct 2024 fix) is removed as part of the rewrite.

`git show d3e503c564`, the earlier fix, is a single-line diff that added `{droppableProvided?.placeholder}` inside `StyledColumnCardsContainer` — confirming the perf commit regressed that exact fix.

Confidence that `703d9cb08d` introduced this regression: very high (0.95). It is the only commit on these files that moves the placeholder, and it does so at the exact location where the current bug lives.

### Phase D — Similar past issues/PRs

Only local `git log` / `git show` were used; no network `gh search` was attempted (per counterfactual gate). Within this repository's history, the `fix: droppable-placeholder` commit (Oct 2024) is the direct analogue — same library, same symptom description, and it was resolved by placing the placeholder inside the `innerRef` element. The fix pattern from that precedent is the authoritative reference for what correct placement looks like.

The reporter cites upstream PR numbers in the issue body as historical context — those are treated as input signals only and are recorded in `leak_signals` below; they are not re-emitted in this analysis.

### Phase E — Synthesis

Root cause (high confidence): `droppableProvided.placeholder` is rendered as a sibling of the `innerRef` DOM node rather than a descendant, violating `@hello-pangea/dnd`'s API contract. When a card is picked up, the placeholder is meant to occupy that card's space inside the `innerRef` container so the column retains its full height during drag; because the placeholder lives in the wrapper `StyledColumn`, the `innerRef` container shrinks by the card's height, and the library computes hit-detection against a bounding box that is shorter than the visible column — so drops beyond that box (anywhere below the midpoint of a long column) are silently ignored. The longer the column, the more pronounced the deadzone, matching the reporter's observation that drops only register near the top.

Why the code looks this way: the perf refactor added a `React.memo` boundary (`DragAndDropLibraryLegacyReRenderBreaker`) around the cards container so that transient drag-state updates from `@hello-pangea/dnd` would not re-render every card. Rendering `droppableProvided.placeholder` inside the memoized component would defeat that memo, because `droppableProvided` is a fresh object on every drag tick. The author moved the placeholder out to preserve the perf win but broke the library contract.

Suggested direction (for Architect, not a binding plan): the placeholder must be placed inside the `innerRef` element, but rendered through a path that does not cause the memoized card-list subtree to re-render each tick. Several approaches worth considering:

- A small non-memoized child of `StyledColumnCardsContainer` whose sole job is to render `droppableProvided.placeholder`, so the card list siblings inside the same container are still wrapped in their own memoization.
- Restructuring so `innerRef` + `droppableProps` + `placeholder` live on a thin wrapper component that is not memoized, and the heavy card list is a memoized child of that wrapper (i.e., move the memo boundary down one level instead of over the placeholder).
- Confirming whether `ignoreContainerClipping` on the `Droppable` is additionally required given the `ScrollWrapper` ancestor; the issue body flags this as a contributing factor but the primary cause is placeholder placement.

Confirm whichever approach preserves the perf gains claimed by the original refactor (only the two reordered columns re-render at drop, not every card).

## Hypothesis

The Nov 2025 Kanban perf refactor wrapped `RecordBoardColumnCardsContainer` in a `React.memo` (`DragAndDropLibraryLegacyReRenderBreaker`) and, as part of the same change, relocated `{droppableProvided.placeholder}` from inside `StyledColumnCardsContainer` (the `innerRef` element) to a sibling position inside the `StyledColumn` wrapper. This violates `@hello-pangea/dnd`'s requirement that the placeholder be a descendant of the `innerRef` node, causing the droppable's measured bounding box to shrink when a card is picked up, which produces the visible "drops only register near the top of the column" symptom on long columns.

## Confidence

**0.92** — root cause is directly visible in current source, the regression commit is pinpointed in local git history, the precedent fix in the same repo (`fix: droppable-placeholder`) confirms the exact placement that works, and `@hello-pangea/dnd`'s documented contract is unambiguous on this point. Not 1.0 because it has not been verified in a running browser and there may be a secondary contribution from `ignoreContainerClipping` interacting with `ScrollWrapper` that only a runtime test would reveal.

## Affected repos

- `ShankHarinath/twenty` — frontend only. Changes are contained to `RecordBoardColumn.tsx` and `RecordBoardColumnCardsContainer.tsx` (and possibly a new tiny sibling component to hold the placeholder without breaking the memo boundary).

## Open questions

- Can the placeholder be moved back inside `StyledColumnCardsContainer` without defeating the `DragAndDropLibraryLegacyReRenderBreaker` memoization? Architect needs to decide between (a) a narrow non-memoized placeholder slot inside the container, (b) restructuring the memo boundary, or (c) another pattern the codebase already uses elsewhere.
- Is `ignoreContainerClipping` on the `Droppable` additionally needed given the `ScrollWrapper` ancestor, or is it sufficient to fix placeholder placement alone? Worth testing after the primary fix lands.
- Do the record-table drag/drop components (which have their own droppable/placeholder split, e.g. `RecordTableBodyDroppablePlaceholder.tsx`) suffer from the same anti-pattern or use a different pattern that could be reused here? Not required for this fix but relevant for code-health follow-up.

## Recommended next step

Hand off to Architect (`/phoenix:plan`) with the two-file edit scope above. The fix is localized and low-risk (GitNexus impact is LOW, 0 d=1 callers affected), but requires a design decision about how to preserve the memoization contract while restoring placeholder placement.

---

```yaml
affected_repos:
  - ShankHarinath/twenty
confidence: 0.92
recommended_next_step: plan
leak_signals:
  - "#15714"      # upstream PR cited in issue body as the suspected regressor
  - "#7600"       # upstream PR cited in issue body as the prior fix
  - "#7597"       # upstream issue cited in issue body as the prior bug report
gate_violations: []
```
