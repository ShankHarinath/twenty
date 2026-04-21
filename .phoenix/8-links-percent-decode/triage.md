# Triage Report — Issue #8

## Summary

URL-type (Links) field values are being silently mutated on write. The shared helper `lowercaseUrlOriginAndRemoveTrailingSlash` — used both on CSV/spreadsheet import in the frontend and on every Links-field GraphQL write on the backend — calls `decodeURIComponent` on the URL's `pathname` and `search` segments. That decodes any percent-encoded reserved characters (notably `%2F` → `/`) inside the path/query, which breaks URLs whose meaning is sensitive to the exact percent-encoding (Google Maps `data=...!16s%2Fg%2F1ptwh8096!...`, signed URLs with encoded slashes, API callback URLs, etc.). The reporter's "working vs broken" URLs differ only in that `%2F` segments are decoded to `/` — the exact behaviour of this helper, and it is already asserted in a passing test.

The reporter's mental model ("happens to all url type fields") is correct: the backend transformer `transformLinksValue` runs for all create/update operations on `FieldMetadataType.LINKS`, not just imports.

## Affected repos

- **ShankHarinath/twenty** — code fix lives here.

## Phase A — Code map

Search radius: only `twenty` is relevant (the issue is internal to the twenty data model; other indexed repos are unrelated services).

### Primary suspect

- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-shared/src/utils/url/lowercaseUrlOriginAndRemoveTrailingSlash.ts`
  - Lines 13–16 build the preserved path as `safeDecodeURIComponent(url.pathname) + safeDecodeURIComponent(url.search) + url.hash`. The two `safeDecodeURIComponent` calls are the bug: they decode `%2F`, `%3F`, `%23`, `%26`, etc. inside path and query, losing information the server/parser on the other end relies on.

### Call sites (all paths that write persisted URLs)

- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/core-modules/record-transformer/utils/transform-links-value.util.ts` lines 48 and 55 — applied to every `primaryLinkUrl` and every `secondaryLinks[i].url` on all Links-field writes. `transformLinksValue` is called from:
  - `record-input-transformer.service.ts:84`
  - `data-arg-processor.service.ts:342`
  These cover both standard REST/GraphQL create/update and the common args data arg processor — i.e., essentially all write paths, not just imports.
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-front/src/modules/object-record/spreadsheet-import/utils/buildRecordFromImportedStructuredRow.ts:188` — applied to `primaryLinkUrl` during CSV/lead imports (the path the reporter hit).
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-front/src/modules/object-record/spreadsheet-import/utils/spreadsheetImportGetUnicityTableHook.ts:107` — used to build a dedup key during import. Comparison-only (not written to the record), but the key will mismatch the stored value unless both sides are consistent.
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/modules/contact-creation-manager/services/create-company.service.ts:74` — applied to a `domainName`. Low practical impact (domain names rarely carry encoded path chars) but will still be wrong for domains with unusual paths.

### Test that codifies the current (buggy) behaviour

- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-shared/src/utils/url/__tests__/lowercaseUrlOriginAndRemoveTrailingSlash.test.ts` lines 78–82:

  ```
  title: 'should handle mixed encoded and non-encoded in same URL',
  input: 'https://example.com/path%2Fwith%2Fslashes?query=hello%20world',
  expected: 'https://example.com/path/with/slashes?query=hello world',
  ```

  This is precisely the transformation that breaks the reporter's Google Maps URL. Any fix must invert this assertion (and adjust the other "decode" cases at lines 47–50 and 64–67 that incidentally exercise the same decoder path).

GitNexus note: `mcp__gitnexus__impact` returned "Corrupted wal file" on this target. Fallback via Grep found all direct call sites listed above.

## Phase B — Production log corroboration

Not applicable / skipped: this is a data-mutation bug whose transformation is fully reproducible from the source, and the reporter provided the exact before/after strings. The `kind-canonix` cluster does not host this product. No log signal required.

## Phase C — Recent commits on affected files

Not pursued in depth (counterfactual gate). The bug's test is long-established and asserts the decoding behaviour, so this is not a recent regression — it's a design defect in the helper. The reporter likely only hit it now because they imported a URL type (Google Maps) that carries semantically-significant `%2F` in its path.

## Phase D — Similar past issues / PRs

Not searched: counterfactual gate forbids upstream lookup, and the fork has no historical issues relevant to this. No leak signals encountered.

## Phase E — Root cause hypothesis

`lowercaseUrlOriginAndRemoveTrailingSlash` intentionally calls `safeDecodeURIComponent` on `url.pathname` and `url.search` (to normalize things like `%C3%A9` → `é` for display/comparison), but `decodeURIComponent` does not distinguish between non-ASCII characters that are safe to decode and reserved characters (`/`, `?`, `&`, `#`, `=`, `+`, `%`, etc.) whose *encoded* form is semantically different from their *decoded* form. For Google Maps `data=...` payloads, signed URLs, OAuth `state` parameters, and any URL that uses `%2F` inside a single path/query segment, decoding destroys the URL.

Correct behaviour is to normalize *only* the origin (lowercase host, strip trailing slash) and leave pathname / search / hash byte-for-byte as the user provided them. If the origin-preserving callers do want to compare "equivalent" URLs ignoring case-only encoding differences (e.g., unicity dedup in the spreadsheet import), that comparison normalization should be a *separate* helper that is used only for comparison — never for the value that is persisted.

**Confidence: 0.9.** Direct source reading plus the existing assertion in the test suite matches the reporter's exact before/after strings.

## Recommended fix (for Architect)

Two-part change, minimally invasive:

1. In `packages/twenty-shared/src/utils/url/lowercaseUrlOriginAndRemoveTrailingSlash.ts`, drop the two `safeDecodeURIComponent` calls. Rebuild the URL using the untouched `url.pathname`, `url.search`, and `url.hash` so that any percent-encoding the user supplied round-trips unchanged. Still lowercase the origin and strip the trailing slash. When `getURLSafely` returns `undefined`, keep the existing early return of `rawUrl`.
2. Update the sibling test file to invert the now-incorrect assertions — specifically the "mixed encoded and non-encoded" case (expect `%2F` to round-trip) and re-check the two cases on lines 47–50 and 64–67 which rely on decoding behaviour. If any legitimate case-insensitive comparison needs case-normalization of percent-encoded bytes, introduce a separate helper used only by comparison paths (e.g., the spreadsheet unicity hook) and never by the persisted-value path.

No migration is needed for existing already-corrupted rows (the helper only runs on write), but a one-time data-fix script to recover mangled URLs is not possible from the mangled value alone — those are genuinely lost and users must re-import.

## Open questions

- Does the product still *want* normalization of percent-encoded non-ASCII (e.g., `%C3%A9` → `é`) for display purposes? If yes, that normalization must move to a display-only layer (not the persistence pipeline). Architect should decide.
- The unicity-dedup call site (`spreadsheetImportGetUnicityTableHook.ts`) currently uses the same helper for comparison; after the fix, import dedup will consider `https://x/a%2Fb` and `https://x/a/b` as different rows. Is that the desired semantics? Likely yes (they *are* different URLs).
- `create-company.service.ts` feeds `domainName` through this helper. Should domain-name normalization be a separate, simpler helper (origin-only)?

## Machine-readable footer

```yaml
affected_repos:
  - ShankHarinath/twenty
confidence: 0.9
recommended_next_step: plan
primary_files:
  - packages/twenty-shared/src/utils/url/lowercaseUrlOriginAndRemoveTrailingSlash.ts
  - packages/twenty-shared/src/utils/url/__tests__/lowercaseUrlOriginAndRemoveTrailingSlash.test.ts
  - packages/twenty-server/src/engine/core-modules/record-transformer/utils/transform-links-value.util.ts
  - packages/twenty-front/src/modules/object-record/spreadsheet-import/utils/buildRecordFromImportedStructuredRow.ts
  - packages/twenty-front/src/modules/object-record/spreadsheet-import/utils/spreadsheetImportGetUnicityTableHook.ts
  - packages/twenty-server/src/modules/contact-creation-manager/services/create-company.service.ts
leak_signals: none
```
