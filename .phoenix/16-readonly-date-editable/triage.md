# Triage Report — ShankHarinath/twenty issue #16

## Summary

`FormDateFieldInput` does not truly enforce `readonly`. It only applies CSS `pointer-events: none` / `user-select: none` on the wrapper `StyledDateInputTextContainer`, and the inner `<DatePickerInput>` renders a plain `<input type="text">` with no `disabled`/`readOnly` HTML attribute. Keyboard focus (Tab, programmatic `.focus()`, Shift+Tab into the field) bypasses CSS `pointer-events`, so users can still type, IMask's `onComplete` still fires, and `handleInputChange` still calls `persistDate(newDate) → onChange(newDate)` — mutating form state on a readonly field. The sibling component `FormDateTimeFieldInput` handles this correctly by passing `readonly` down to `DateTimePickerInput`, which then renders `<input disabled={shouldDisplayReadOnly}>`. The date variant was regressed during refactor PR #19210 which replaced the old `<StyledDateInput disabled>` implementation with the reused `DatePickerInput` wrapper that lacks a readonly prop.

## Phase A — Code mapping

**Repos searched:** `twenty` only. The home repo is a standalone React/TypeScript frontend monorepo; no group membership for this issue (no cross-service contract involved — this is purely a client-side form input bug). No other indexed repos contain `FormDateFieldInput` or `DatePickerInput` symbols.

**Signal → symbol mapping:**

| Signal | Symbol | File | Lines |
|---|---|---|---|
| `FormDateFieldInput` | `FormDateFieldInput` | `packages/twenty-front/src/modules/object-record/record-field/ui/form-types/components/FormDateFieldInput.tsx` | 77–309 |
| `handleInputChange` | `handleInputChange` | same file | 180–192 |
| `DatePickerInput` | `DatePickerInput` | `packages/twenty-front/src/modules/ui/input/components/internal/date/components/DatePickerInput.tsx` | 47–114 |
| `IMask` / `onComplete` | `useIMask(...).onComplete` callback → calls `onChange(parsedDate)` | `DatePickerInput.tsx` | 89–93 |
| CSS `pointer-events` | `StyledDateInputTextContainer` (linaria styled-div) | `FormDateFieldInput.tsx` | 35–50 |
| Readonly gate | Prop `readonly` passed only to wrapper div `isReadonly={readonly === true}`; NOT forwarded to `<DatePickerInput>` | `FormDateFieldInput.tsx` | 267–273 |

**Key code excerpts (the bug surface):**

- `FormDateFieldInput.tsx:266-274` — renders `<DatePickerInput date={...} onChange={handleInputChange} />` with no `readonly` / `disabled` prop passed through. The wrapper's `pointer-events: none` does not block keyboard events.
- `FormDateFieldInput.tsx:180-192` — `handleInputChange` unconditionally calls `setDraftValue(... mode: 'edit')` and `persistDate(newDate)`; it never checks `readonly`.
- `DatePickerInput.tsx:104-112` — renders `<StyledInput type="text" ref={ref as any} value={value} onChange={() => {}} />`. No `disabled`, no `readOnly`, no IMask `readonly` option.
- `DatePickerInput.tsx:42-45` — prop type is `{ onChange?: ...; date: string | null }`; it does not accept a readonly prop at all.

**Contrast with working sibling** (`FormDateTimeFieldInput.tsx:256` → `<DateTimePickerInput readonly={readonly} ...>` → `DateTimePickerInput.tsx:164,172` sets `const shouldDisplayReadOnly = readonly === true; ... disabled={shouldDisplayReadOnly}`). This proves the intended pattern and confirms the date-only variant is the outlier.

**Impact analysis:** `impact(FormDateFieldInput, upstream) = 0 direct callers / LOW risk` and `impact(DatePickerInput, upstream) = 0 direct callers / LOW risk` — the GitNexus graph does not register the JSX usages as CALLS edges, but a grep confirms `DatePickerInput` is only imported in `FormDateFieldInput.tsx` (all other matches are locale `.po` files, an unrelated `SettingsDatePickerInput`, and utility hooks). `FormDateFieldInput` is referenced by workflow form rendering paths but the `readonly` prop contract is already part of the public signature, so fixing the internal plumbing is non-breaking.

## Phase B — Runtime confirmation

Skipped intentionally. This is a client-side React UI correctness bug triggered by keyboard focus on a specific form field; it produces no distinctive server log signature. There is no error thrown, no HTTP error, and no backend persistence failure — the bug is that `onChange` IS called when it should not be. `kubectl logs` against `kind-canonix` would not corroborate or refute the claim. The reproduction is deterministic via the test snippet in the issue body (focus + IMask `onComplete`).

Noted as an open question: no production telemetry is available to gauge real-world frequency, but given workflow forms frequently use readonly date fields on published-but-not-editable steps, any non-zero keyboard usage on these fields would silently corrupt form state.

## Phase C — Recent commits on affected files

```
19dd4d6c1b Fix workflow date fields (#19210)  [Apr 1 2026]  ← root-cause commit
cac4999e9f fix: handle Escape in date/datetime pickers and remove ValidationStep any (#18107)
ef499b6d47 Re-enable disabled lint rules and right-size CI runners (#18461)
c53a13417e Remove all styled(Component) patterns in favor of parent wrappers and props (#18430)
7a2e397ad1 Complete linaria migration (#18361)
0b5be7caa3 Refactored Date to Temporal in critical date zones (#16544)
```

**PR #19210 (`19dd4d6c1b`, "Fix workflow date fields")** is the regression. The diff (seen via `git show 19dd4d6c1b -- packages/twenty-front/.../FormDateFieldInput.tsx`) shows:

- The previous implementation used a local `<StyledDateInput ... disabled={readonly}>` with an explicit `&:disabled` CSS branch — the HTML `disabled` attribute reliably blocked keyboard input and IMask mutations.
- PR #19210 replaced that with `<DatePickerInput date onChange={handleInputChange} />` wrapped in `<StyledDateInputTextContainer isReadonly>` that only styles via CSS `pointer-events: none`.
- The `placeholder` prop was also dropped in the refactor (separate minor issue, not part of this bug).
- The PR's stated motivation ("simplifying FormDateInput using the existing DatePicker") made no mention of the readonly contract, which appears to have been overlooked. The companion `FormDateTimeFieldInput` in the same PR kept readonly correct only because it uses the separate `DateTimePickerInput` that already had a `readonly` prop.

Correlation confidence: high. The bug description exactly matches the post-refactor code, and the pre-refactor code was known-correct.

## Phase D — Similar past issues/PRs

`gh` access to the upstream `twentyhq/twenty` org is blocked by the harness ACL (`gh-shim: DENIED — owner 'twentyhq' not in allowlist`), so I could not search GitHub for prior readonly/date issues. From the local commit history, the closest analogous fix pattern is the one already applied in `DateTimePickerInput`: thread `readonly` down, compute `shouldDisplayReadOnly`, render `disabled={shouldDisplayReadOnly}`, and guard the `onChange` / IMask completion callback. That pattern should be replicated in `DatePickerInput` and `FormDateFieldInput`.

Noted open question: if the reporter's comment on PR #19208 was deleted (per the issue body), upstream may have additional context not captured here.

## Phase E — Root cause hypothesis and fix sketch

**Hypothesis.** During the refactor in PR #19210, the readonly enforcement regressed from HTML-level (`<input disabled>`) to CSS-level (`pointer-events: none` on a wrapper div). CSS `pointer-events` only blocks pointer/mouse events, not keyboard focus or keyboard input. A keyboard-focused `<input>` inside a `pointer-events: none` ancestor still receives `keydown`/`keyup`/`input` events; IMask listens to these and fires `onComplete(newValue)` which calls `FormDateFieldInput.handleInputChange`, which unconditionally persists the new date. The fix requires (a) forwarding `readonly` into `DatePickerInput`, (b) rendering `<input disabled={readonly}>` there, and (c) defensively guarding `handleInputChange` in the parent with an `if (readonly) return;` so the invariant is enforced in two places (belt-and-suspenders mirroring `handleMaskedDatePointerDownCapture` at line 243 which DOES check readonly).

**Minimal fix outline** (for Architect):

1. `DatePickerInput.tsx`: add `readonly?: boolean` to `DatePickerInputProps`; pass `disabled={readonly}` onto `<StyledInput>`; optionally also short-circuit the `onComplete` handler when `readonly` is true (safety).
2. `FormDateFieldInput.tsx`: pass `readonly={readonly}` into `<DatePickerInput>` at line ~270; add `if (readonly) return;` as the first line of `handleInputChange` (line 180).
3. Optional hardening: keep the existing `pointer-events`/`user-select` CSS (still useful for pointer affordances and visual cue) but do not rely on it as the enforcement layer.

**Confidence: 0.90.** The bug is fully visible in source, the regressing commit is identified, the repair pattern already exists in the sibling component, and the reproduction recipe in the issue body is deterministic. The only residual uncertainty (hence not 0.95) is whether additional call-sites of `DatePickerInput` exist that I missed (grep shows only one, but linting/rendering edge-cases could surface) and whether IMask needs its own `readonly: true` option to fully suppress autofix behavior — worth validating in the implementation PR but not a blocker.

## Open questions / blockers

- `gh` access to `twentyhq/twenty` is blocked by harness ACL, so upstream issue/PR search was not possible. The reporter mentions their comment on PR #19208 was deleted; if Architect has upstream access, worth checking whether the fix has already been proposed.
- No runtime log signal (Phase B skipped by design for a client-side UI correctness bug).
- A pre-existing test file for `FormDateFieldInput` (if any) should be audited — it likely did not assert the readonly-with-keyboard-focus case; the new fix should add that test per the reporter's exact recipe.

## Recommended next step

Hand to Architect for a focused patch. The change is local (two files, <20 lines), has no cross-repo implications, risk is LOW per GitNexus impact, and the intended pattern is already present in the sibling `DateTimePickerInput` / `FormDateTimeFieldInput` pair to serve as the reference. Ship with a Vitest/React Testing Library test that programmatically focuses the input, types a full date string to fire IMask `onComplete`, and asserts `onChange` was NOT called when `readonly={true}`.

## Files referenced (absolute paths)

- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-front/src/modules/object-record/record-field/ui/form-types/components/FormDateFieldInput.tsx`
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-front/src/modules/ui/input/components/internal/date/components/DatePickerInput.tsx`
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-front/src/modules/object-record/record-field/ui/form-types/components/FormDateTimeFieldInput.tsx` (reference implementation, no change required)
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-front/src/modules/ui/input/components/internal/date/components/DateTimePickerInput.tsx` (reference implementation, no change required)

---

```yaml
# machine-readable footer
affected_repos:
  - ShankHarinath/twenty
confidence: 0.90
recommended_next_step: proceed
```
