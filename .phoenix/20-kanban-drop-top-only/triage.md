## Triage Report — ShankHarinath/twenty#20

**Classification:** bug  (orchestrator confidence: 0.95)
**Summary:** `@hello-pangea/dnd` placeholder is rendered outside the `innerRef` element on Kanban columns, shrinking the droppable's measured bounding box by the dragged card's height so only drops near the top of a column register.

### Symptom

- Normalized: When dragging a card between Kanban columns, the drop target only accepts drops near the top of the destination column; the middle and bottom of the column appear inert.
- Error signatures: none (pure behavior bug — drops silently fail, no console error, no server log)
- Reported timeframe: regression introduced Nov 2025 (PR #15714 merged 2025-11-11)
- Affected entities: any user on a Kanban/board view with at least one column tall enough to scroll

### Runtime signal (from logs.search)

- First-seen: **N/A — client-side rendering/measurement bug, no backend signal**
- Frequency: n/a
- Active now: unknown (no backend observability for client hit-testing)
- Service state: n/a
- Target(s): n/a — bug lives in `twenty-front` React code
- Backend source: `logs.search` not invoked; no service emits a log for this failure mode
- Correlated errors: none expected
- Trace excerpts: none — the library's own drag-lifecycle events fire a `DragDropContext.onDragEnd` with `result.destination = null` when the drop lands outside the measured zone, which is correct from the library's perspective. The silent-failure surface matches a geometry mismatch, not an error.

### Dataflow diagram

```
user drags card across columns
  │
  ▼
DragDropContext (hello-pangea/dnd)       [library root]    RecordBoardDragDropContext.tsx (via RecordBoard.tsx:50)
  │ library measures each <Droppable>'s innerRef bbox
  │
  ▼
Droppable(droppableId=<columnId>)        [column wrapper]  RecordBoardColumn.tsx:66
  │ render prop passes droppableProvided to children
  │
  ▼
StyledColumn (plain <div>)               [styled wrapper]  RecordBoardColumn.tsx:68
  │   ├── DragAndDropLibraryLegacyReRenderBreaker  [memo]   DragAndDropReRenderBreaker.tsx:9
  │   │      └── RecordBoardColumnCardsContainer   [cards]  RecordBoardColumnCardsContainer.tsx:31
  │   │             StyledColumnCardsContainer ref=innerRef  :48–49     ◀── library reads this bbox
  │   │             {cards...}
  │   └── {droppableProvided.placeholder}                   RecordBoardColumn.tsx:77    ◀── defect site
  │                                                         (sibling of innerRef, not descendant)
  ▼
library hit-test uses innerRef bbox        ◀── symptom: bbox shrinks by dragged card height;
                                                only top strip of visual column remains hit-testable
```

Every arrow is a code-level hop. No `[unread]` hops — all participating symbols are in local files.

### Cause-flow diagram

```
reported symptom: "Kanban drops only register near the top of a target column; middle/bottom are inert"

layer                            state at this layer                        observation
─────                            ────────────────────                       ──────────────
user                             drags card from column A into column B     normal
   │
DragDropContext                  receives pointer events                    correct
   │
Droppable(column B) render       droppableProvided = {innerRef, droppable-  correct — library API
   │                             Props, placeholder}                        contract met at this layer
   ▼
StyledColumn children layout     <memo-wrapper>            ← latent: memoization boundary will block
   │                              └─ <cards container with  future placeholder prop updates from
   │                                 ref=innerRef, N cards> reaching the cards container
   │                              <placeholder sibling>
   ▼
Drag starts — card detaches      card leaves document flow inside cards    cards container natural height
   │                             container; placeholder (intended to        drops by card height;
   │                             reserve space inside innerRef container)   placeholder expands in
   │                             is in the SIBLING, not inside              StyledColumn layout instead
   │                                                                        ← bug: placeholder and innerRef
   │                                                                          are in different DOM parents,
   │                                                                          violating hello-pangea/dnd's
   │                                                                          droppable contract
   │                                                                          (RecordBoardColumn.tsx:77)
   ▼
library measures innerRef bbox   bbox = visible column height MINUS         measured droppable region is
   │                             dragged-card height                        smaller than the visual column
   │
   ▼
pointer hit-test                 drop only accepted when pointer y <=       ← symptom: drops near top OK;
                                 top + (visible_height - card_height)       drops in middle/bottom ignored
                                                                            (worse as column height grows)
```

**Reconciliation:** Dataflow defect-site = `RecordBoardColumn.tsx:77` (placeholder placement). Cause-flow `← bug:` marker = same file:line. **Match.** The surface symptom in the cause-flow (shrunken bbox measurement) happens inside the vendor library — the only code under our control that causes it is the placeholder's placement, which is the defect-site in the dataflow.

### Affected code (from codebase.*)

- **Repos searched:** `twenty` (home repo; no cross-repo signals; Kanban UI is fully contained in `twenty-front`)
- **Affected repos (fix required in):** `ShankHarinath/twenty`
- `twenty` · `RecordBoardColumn` · `packages/twenty-front/src/modules/object-record/record-board/record-board-column/components/RecordBoardColumn.tsx:66–80` — Droppable wrapper whose render prop places `{droppableProvided.placeholder}` outside the `innerRef` element (line 77).
- `twenty` · `RecordBoardColumnCardsContainer` · `packages/twenty-front/src/modules/object-record/record-board/record-board-column/components/RecordBoardColumnCardsContainer.tsx:31–80` — owns the `StyledColumnCardsContainer` that receives `droppableProvided?.innerRef` (line 49) and `droppableProps` (line 51). This is the container whose bbox the library measures.
- `twenty` · `DragAndDropLibraryLegacyReRenderBreaker` · `packages/twenty-front/src/modules/ui/drag-and-drop/components/DragAndDropReRenderBreaker.tsx:9–20` — `React.memo` with compare key `memoizationId === memoizationId`; this is the memoization layer that PR #15714 introduced and is the reason the placeholder was relocated. Any fix that reintroduces the placeholder inside the cards container must preserve the memoization benefit.
- `twenty` · `RecordBoard` · `packages/twenty-front/src/modules/object-record/record-board/components/RecordBoard.tsx:36–61` — wraps the board in `ScrollWrapper` (line 42) and `RecordBoardDragDropContext` (line 50). No `ignoreContainerClipping` prop on the per-column Droppable; secondary exposure if scroll clipping aggravates the hit-zone.
- Blast radius (codebase.impact on `RecordBoardColumnCardsContainer`, depth 2): 0 direct callers, 0 transitive — risk `LOW`. This component is used only from `RecordBoardColumn` (single call-site), which is the column primitive for the board. A fix touching either file is contained.
- Cross-repo dependencies: none; `@hello-pangea/dnd` is a third-party lib, not a Canonix repo.
- Process participation: none detected in the graph — the board renders are a React-tree-local concern, not a multi-service process.
- Sibling scan hits (intra-file / cross-package / community / codegen):
  - `packages/twenty-front/src/modules/navigation-menu-item/edit/side-panel/components/SidePanelNewSidebarItemMainMenu.tsx:65, 153` — same library, correct pattern (`{placeholder}` is the last child of the same `<div ref={innerRef}>`). Confirms the expected shape.
  - `packages/twenty-front/src/modules/object-record/record-calendar/month/components/RecordCalendarMonthBodyDay.tsx:150–161` — same library, placeholder on line 161 inside same element as `innerRef` on line 151. Correct pattern. Suggests calendar view is not affected.
  - `packages/twenty-front/src/modules/page-layout/widgets/fields/components/FieldsConfigurationGroupEditor.tsx:181, 213` — same library, correct pattern. Not affected.
  - **Conclusion:** the bug is localized to the record-board module; no other `@hello-pangea/dnd` consumer in the repo exhibits the same placement.
- Misses: no misses — all pre-extracted signals map to code.

### Historical context

**Similar past issues / PRs** (from issues.search + vcs.pr_search):
- `#7597` / PR `#7600` (2024-10-13, Harsh Singh) — *"fix: droppable-placeholder"* — the ORIGINAL fix for this exact bug. Added a single line `{droppableProvided?.placeholder}` inside `StyledColumnCardsContainer` in `RecordBoardColumnCardsContainer.tsx`. One-file, one-line change. This is the canonical "fix shape" for the problem in this codebase.
- PR `#19005` (2026-03-26, Lucas Bordeau) — *"Fix board drag and drop issues"* — addressed adjacent dnd problems (infinite loops, warnings, a Sentry error) but did NOT restore the placeholder inside the `innerRef` element; touched `RecordBoardColumnCardsContainer.tsx` only for an import reorder. The bug persisted through this PR.
- PR `#11781` (2025, `ea25498625`) — *"Fix dragging behavior below the last card when dragging below the new CTA button"* — an earlier drop-below-last-card fix; related touchpoint in the same column render path, not the same symptom.

**Recent commits on affected files** (from `git log`):
- `703d9cb08d` 2025-11-11 Lucas Bordeau — *"Improved Kanban performance (#15714)"* — **the regression commit.** Diff confirms: removed `{droppableProvided?.placeholder}` from `RecordBoardColumnCardsContainer.tsx` (line 122 pre-patch) and added `{droppableProvided.placeholder}` to `RecordBoardColumn.tsx:77` as a sibling of the memo-wrapped cards container. Motive: avoid re-rendering the memoized `RecordBoardColumnCardsContainer` on every placeholder state update. PR body acknowledges remaining perf issues: *"some performance issues at load time and drop, but they are harder to address."*
- `8ef99671e2` 2026-03-26 — `#19005` noted above.
- `d3e503c564` 2024-10-13 Harsh Singh — `#7600` noted above (the prior fix).

**What else did the regression-commit author change** (PR #15714 same-PR siblings):
- `RecordBoard.tsx` (−171 LOC, refactored) — pulled `DragDropContext` into a separate `RecordBoardDragDropContext.tsx`; not directly related to placeholder placement.
- `RecordBoardClickOutsideEffect.tsx` (new, +51) — click-outside for selection; unrelated.
- `RecordBoardDragDropContext.tsx` (new, +98) — moves DragDropContext provider; the `DragDropContext` layer is fine.
- `RecordBoardDragSelect.tsx` (new, +46) — drag-to-select; unrelated to inter-column drops.
- `useSetRecordBoardRecordIds.ts` (−85) — state plumbing; unrelated.
- Introduced `DragAndDropLibraryLegacyReRenderBreaker` (`memo` wrapper) — the memoization boundary that motivated the placeholder move. **Any fix must reconcile with this wrapper.**

### Hypotheses considered

Scout explored the following hypotheses in Phase E1. **Confidence gate fired:** H1 had `root_cause_confidence` = 0.92 with all pre-extracted signals mapped to code and a single regression commit explaining the transition from working to broken — multi-hypothesis investigation (E2) skipped.

- **H1 — `droppableProvided.placeholder` rendered as sibling of the `innerRef` element violates `@hello-pangea/dnd` contract, shrinking the measured bbox** · subagent: `library_contract_checker` · result: confirmed · final_conf: 0.92
  - Falsifiable trace outcome: read both files → innerRef is on `StyledColumnCardsContainer` at `RecordBoardColumnCardsContainer.tsx:49`; placeholder is on `StyledColumn` at `RecordBoardColumn.tsx:77`; they are not in the same subtree. The `git show 703d9cb08d` diff shows PR #15714 removed the placeholder from the innerRef container and added it here. The prior fix (PR #7600, commit `d3e503c564`) added the placeholder to the innerRef container.
  - Key evidence: `RecordBoardColumn.tsx:77`, `RecordBoardColumnCardsContainer.tsx:49`, commit `703d9cb08d` (regression), commit `d3e503c564` (canonical prior fix).
- **H2 — missing `ignoreContainerClipping` on `Droppable` combined with `ScrollWrapper` clips drop zones to visible viewport, amplifying the shrunken-bbox problem** · subagent: `library_contract_checker` · result: inconclusive · final_conf: 0.35
  - Falsifiable trace outcome: `git grep ignoreContainerClipping` returns zero matches in the frontend; default is `false`. Could be a contributing factor when columns scroll, but not the root cause — the reporter shows the bug reproduces in columns that fit the viewport. Worth considering as a hardening option alongside H1.
  - Key evidence: `git grep` zero hits for `ignoreContainerClipping`; `RecordBoard.tsx:42` wraps board in `ScrollWrapper`.
- **H3 — `DragAndDropLibraryLegacyReRenderBreaker` memo staleness prevents the cards container from seeing updated `droppableProvided`, so even restoring placeholder inside the cards container would render a stale placeholder** · subagent: `hoc_decorator_tracer` · result: inconclusive · final_conf: 0.40
  - Falsifiable trace outcome: `DragAndDropReRenderBreaker.tsx:17–19` compares only `memoizationId` (= columnId), so a mid-drag re-render from the Droppable parent is blocked. This is a real constraint on the fix shape, not the root cause itself; it explains why PR #15714 moved the placeholder out in the first place. A correct fix must either (a) bypass memoization for a thin slot dedicated to the placeholder, (b) include the placeholder identity in the memo comparison, or (c) remove the memo boundary and accept the perf regression PR #15714 tried to solve.
  - Key evidence: `DragAndDropReRenderBreaker.tsx:9–20`.

**Consolidator synthesis:** `primary_causal_symbol` = `packages/twenty-front/src/modules/object-record/record-board/record-board-column/components/RecordBoardColumn.tsx:77`; 1 confirmed (H1), 0 refuted, 2 inconclusive (H2, H3 treated as amplifiers / fix constraints, not independent causes). Merges: none. Contradictions: none. Converged: true.

### Root-cause hypothesis

The library `@hello-pangea/dnd` measures each `Droppable`'s effective drop region from the DOM node that receives `droppableProvided.innerRef`, and its design contract requires `droppableProvided.placeholder` to render **inside** that same DOM node so that the placeholder reserves the space vacated by the dragged card and the bbox stays constant. In `RecordBoardColumn.tsx:77` the placeholder is rendered as a sibling of the cards container (inside `StyledColumn`), not inside `StyledColumnCardsContainer` (which holds `innerRef`). During a drag, the cards container's natural height drops by the dragged card's height while the placeholder expands in the parent's flexbox instead. The library's hit-test then sees a droppable region that is one-card shorter than the visually painted column. Drops at the top still fall inside both the visible and measured regions and work; drops in the middle or bottom fall inside the visible region but outside the measured region and are silently rejected with `result.destination = null`. The regression was introduced by PR #15714 (commit `703d9cb08d`, 2025-11-11), which moved the placeholder out of the memoized `RecordBoardColumnCardsContainer` to avoid invalidating `DragAndDropLibraryLegacyReRenderBreaker` on every placeholder update during a drag — a performance optimization that broke the library contract. PR #7600 (commit `d3e503c564`, 2024-10-13) is the canonical prior fix and demonstrates the single-line corrective shape (`{droppableProvided?.placeholder}` as the last child of `StyledColumnCardsContainer`).

**Root-cause confidence:** 0.92
**Fix-direction confidence:** 0.70

The fix-direction confidence is a notch lower than root-cause because the naive one-line restoration of the placeholder inside `RecordBoardColumnCardsContainer` will re-subject that subtree to memoized-re-render invalidation on every placeholder update during a drag — re-introducing (part of) the perf problem PR #15714 solved. A robust fix needs to render the placeholder inside the `innerRef` element while keeping the memo boundary effective; see Fix shapes below for the three variants.

### Evidence chain

1. `RecordBoardColumn.tsx:66–80` renders `<Droppable>` where the render prop builds `<StyledColumn>` with a memoized `RecordBoardColumnCardsContainer` followed by `{droppableProvided.placeholder}` as a sibling at `:77`. The `innerRef` is NOT on `StyledColumn` — it is forwarded to `StyledColumnCardsContainer` inside the cards container. → The placeholder and innerRef live in different DOM parents. [`packages/twenty-front/src/modules/object-record/record-board/record-board-column/components/RecordBoardColumn.tsx:66–80`]
2. `RecordBoardColumnCardsContainer.tsx:48–52` attaches `ref={droppableProvided?.innerRef}` and spreads `droppableProvided?.droppableProps` onto `StyledColumnCardsContainer`. The placeholder is NOT rendered inside this container. → The DOM element the library measures shrinks by the dragged card's height once dragging starts. [`packages/twenty-front/src/modules/object-record/record-board/record-board-column/components/RecordBoardColumnCardsContainer.tsx:48–52`]
3. `git show 703d9cb08d` on both files shows PR #15714 (a) removed `{droppableProvided?.placeholder}` as the last child of `StyledColumnCardsContainer`, and (b) added `{droppableProvided.placeholder}` as a sibling of the memo-wrapped container in `RecordBoardColumn.tsx`. → The regression is pinpointed to this commit. [`sha 703d9cb08d`]
4. `git show d3e503c564` on PR #7600 shows the identical issue was fixed before by adding a single line `{droppableProvided?.placeholder}` as the final child of `StyledColumnCardsContainer`. → The canonical prior fix in this codebase is a one-line placeholder insertion inside the innerRef container. [`sha d3e503c564`]
5. `DragAndDropReRenderBreaker.tsx:9–20` — `memo` comparator is `prev.memoizationId === next.memoizationId`. With `memoizationId = recordBoardColumnId` (constant for a given column), the memo blocks ALL re-renders of the cards container during a drag. → Restoring the placeholder inside the cards container without further changes would render a stale `droppableProvided.placeholder` value — the bug might re-emerge differently, or at minimum the fix must account for the memo. [`packages/twenty-front/src/modules/ui/drag-and-drop/components/DragAndDropReRenderBreaker.tsx:9–20`]
6. `git grep droppableProvided.placeholder -- "packages/twenty-front/src/modules/object-record/record-board/**/*.tsx"` returns one hit — `RecordBoardColumn.tsx:77`. `git grep` across all `@hello-pangea/dnd` consumers in `twenty-front` confirms every other consumer renders `{placeholder}` inside the same element that holds `innerRef`. → The bug is localized to record-board; no other consumer has the same structural defect. [`git grep` sibling scan]
7. `git grep ignoreContainerClipping` returns zero hits in `twenty-front`. → Reporter's contributing-factor hypothesis about scroll clipping is plausible in principle but unverified; even without scroll clipping the placeholder-placement defect explains the symptom fully. [`git grep`]
8. `codebase_impact({name: "RecordBoardColumnCardsContainer", depth: 2})` returns `affected = 0`, risk `LOW`. → Fix blast radius is localized; either of the two files can be safely edited without concern for transitive call-site breakage. [`codebase_impact`]

### Scope limits — what this fix does NOT handle

- **`ignoreContainerClipping` scroll-clipping edge cases.** When a column is long enough that cards are scrolled out of view and users drag near the clipped edge, even a correctly-placed placeholder may not fully restore the hit zone without adding `ignoreContainerClipping` on the `Droppable`. This is the reporter's "contributing factor" and is out of scope for the single-line-style fix.
- **Perf regression introduced by placing placeholder inside memo boundary.** If the fix simply moves `{droppableProvided.placeholder}` back into `RecordBoardColumnCardsContainer`, the cards container will need to re-render during drag — (partially) undoing PR #15714's memoization benefit. The minimal fix restores correctness at the cost of some perf; only the Idiomatic/Architectural variants below both restore correctness AND preserve perf.
- **Drops at the very bottom below the `new-<col>-bottom` disabled `Draggable`.** `RecordBoardColumnCardsContainer.tsx:63–75` renders a disabled `Draggable` below the cards and a `StyledNewButtonContainer` below that. Depending on where the placeholder is placed (above or below these siblings inside the container), drops very close to the bottom may still behave unexpectedly. This is the same area PR #11781 touched; the fix should keep the placeholder as the final child of the cards container, matching PR #7600's shape.
- **Other `@hello-pangea/dnd` droppables in the app.** The fix only covers the Kanban board. Other consumers already follow the correct pattern, so no cross-bleed, but a programmatic lint rule to prevent regressions is not included.

### Fix shapes (pragmatism axis)

- **Minimal (parsimonious):** In `RecordBoardColumnCardsContainer.tsx`, add `{droppableProvided?.placeholder}` as the final child of `StyledColumnCardsContainer` (immediately before its closing tag, or specifically as the last child before `<StyledNewButtonContainer>` to match PR #7600's shape). In `RecordBoardColumn.tsx`, remove `{droppableProvided.placeholder}` from line 77. This matches PR #7600 exactly. Trade-off: the memo on `RecordBoardColumnCardsContainer` no longer insulates the cards container from placeholder-state churn during a drag, since the placeholder now lives inside the memoized subtree; but because the memo comparator uses only `memoizationId`, the memo still blocks those re-renders — which means the placeholder inside will be STALE and the bug may manifest in a new form (placeholder not expanding during drag). **This "pure minimal" is unsafe in this codebase** because of the memo. A truly minimal safe variant is to also remove the `DragAndDropLibraryLegacyReRenderBreaker` wrapper, accepting PR #15714's perf regression. Est. LOC: ~5 (remove wrapper + move placeholder) · files: 2.
- **Idiomatic (recommended):** Keep the memo wrapper for the card-list subtree, but render the placeholder through a thin non-memoized slot that lives inside the `innerRef` element. Concretely: in `RecordBoardColumnCardsContainer`, accept `droppableProvided` as prop (already does) and render `<StyledColumnCardsContainer ref={droppableProvided.innerRef} {...droppableProvided.droppableProps}>` containing (a) a memoized `<RecordBoardColumnCards memoizationId={columnId} />` subcomponent that owns only the cards list + fetch loader (no reference to `droppableProvided`), and (b) `{droppableProvided.placeholder}` as a direct (non-memoized) child of `StyledColumnCardsContainer`. Move `DragAndDropLibraryLegacyReRenderBreaker` to wrap only the cards list, not the whole container. This gives the library its required DOM shape AND preserves the re-render-breaking benefit. Mirrors the pattern the SidePanel droppables already follow. Est. LOC: ~30–50 · files: 2–3 (restructure `RecordBoardColumnCardsContainer.tsx`, possibly introduce a new `RecordBoardColumnCards.tsx`, update `RecordBoardColumn.tsx` to drop its placeholder sibling).
- **Architectural (maximal):** Replace the `DragAndDropLibraryLegacyReRenderBreaker` workaround entirely — either migrate the Kanban board to `@dnd-kit/core` (a newer library without the forced-re-render contract, already present in the repo as used in `CommandMenuAddToNavDroppableDndKit.tsx`), or invest in a holistic virtualization + memoization strategy for the board that doesn't rely on ad-hoc `React.memo` wrappers whose comparators exclude critical drag-state. The `DragAndDropReRenderBreaker.tsx` file already carries a `@deprecated` note pointing in this direction. Est. LOC: ~500–1500 · files: 20+.

**Recommended:** **Idiomatic** — `codebase_impact` returned LOW, so Minimal is not blocked by risk, but the Minimal form as literally written is unsafe due to the memo wrapper (stale-placeholder risk). The Idiomatic variant restores the library contract at the correct DOM layer AND preserves the perf win of PR #15714, at a small restructuring cost. The Architectural variant is the right long-term direction and is consistent with the deprecation note on `DragAndDropLibraryLegacyReRenderBreaker`, but it is a much larger project and not warranted as an immediate bug fix.

### Maintainer-review self-pass

1. "Does the restored placeholder inside the memoized subtree re-introduce the perf issue PR #15714 solved?" — Anticipate this; answer by showing the Idiomatic fix-shape where the placeholder is a direct (non-memoized) child of `StyledColumnCardsContainer` while the cards list stays inside `DragAndDropLibraryLegacyReRenderBreaker`. Include a perf note or screencapture of column re-render counts before/after.
2. "Should we also set `ignoreContainerClipping` on the `Droppable`?" — Mention the contributing-factor hypothesis; recommend testing with columns much taller than the viewport and, if clipping still truncates the hit zone, add `ignoreContainerClipping` as a secondary hardening. Out of scope for the primary correctness fix but should be filed as a follow-up.
3. "Why not fix this by migrating to `@dnd-kit/core` (like `CommandMenuAddToNavDroppableDndKit.tsx`)?" — Acknowledge `@deprecated` note on `DragAndDropLibraryLegacyReRenderBreaker`; explain this is the Architectural variant and out of scope for a regression fix; propose a follow-up tech-debt ticket.

### Devil's advocate — if the reporter's theory is wrong

Three alternative causal sites: (a) **`RecordBoardDragDropContext` misconfiguration** — if `DragDropContext` were misusing `onDragEnd` / `onDragUpdate` handlers, drops could fail at the event layer rather than the geometry layer; but the symptom is location-dependent (top works, bottom doesn't), which points at geometry, not event handling. (b) **Stale jotai/recoil state for `recordIndexRecordIdsByGroup`** during a drag, causing the cards list to re-render with different lengths and destabilize library measurement; possible, but this would manifest as intermittent failures rather than the consistent "top-only-works" pattern. (c) **CSS `transform` or `overflow` on an ancestor creating an unexpected containing block or scroll viewport** — `@hello-pangea/dnd` is sensitive to these; `RecordBoard.tsx`'s `ScrollWrapper` is a candidate, and `StyledColumn` has `position: relative` (not problematic by itself). Worth spot-checking with DevTools during a drag if the placeholder fix does not fully resolve the symptom; but the placeholder-misplacement theory already explains the symptom completely and is directly supported by the regression-commit diff, so neither alternative rises to comparable confidence.

### Recommended next step

**proceed to fix**

### Open questions

- Does the Idiomatic fix-shape (non-memoized placeholder sibling inside `StyledColumnCardsContainer` + memo scoped to only the cards list) fully restore the perf gain of PR #15714, or only partially? Verify empirically (column re-render counts during a drag before/after).
- Is `ignoreContainerClipping` needed once the placeholder is correctly placed? Reproduce with a very tall column that forces intra-column scrolling and confirm drops register at the bottom post-fix.
- Are there other `Droppable`s elsewhere in the app (outside the grep patterns I surveyed) that follow the broken shape? A light lint rule or codemod would prevent regressions but is out of scope for this fix.

<!-- MACHINE-READABLE FOOTER — DO NOT REMOVE; downstream skills parse this block -->
<!--phoenix:scout-summary
affected_repos: [ShankHarinath/twenty]
root_cause_confidence: 0.92
fix_direction_confidence: 0.70
confidence: 0.92
recommended_next_step: proceed
primary_causal_symbol: packages/twenty-front/src/modules/object-record/record-board/record-board-column/components/RecordBoardColumn.tsx:77
fix_shape_recommended: idiomatic
hypotheses_count: 3
hypotheses_confirmed: 1
converged: true
-->
