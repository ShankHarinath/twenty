# Triage Report — ShankHarinath/twenty#1

## Summary

The `neq` operator in `compute-where-condition-parts.ts` emits `field != :val OR field IS NOT NULL` when the field has a Postgres default null-equivalent value (e.g. empty string for TEXT). This is logically wrong. Because essentially all non-nullable TEXT columns in Twenty have an empty-string default, `field IS NOT NULL` evaluates to `true` for virtually every row, causing the entire predicate to collapse to `true` — i.e., `neq` filters stop excluding anything on TEXT fields and composite sub-fields (firstName, lastName, primaryEmail, primaryPhoneNumber, address sub-fields, etc.).

By De Morgan's law, the negation of the `eq` predicate `field = :val OR field IS NULL` is `field != :val AND field IS NOT NULL`. The fix is to replace the single `OR` with `AND` on line 68. No other operator is affected.

## Phase A — Code map

### A.1 Search radius
- Indexed repos: 11 (canonix-* group, erpnext, twenty, mattermost, etc.).
- Issue explicitly names `compute-where-condition-parts.ts` in the `ShankHarinath/twenty` repo (indexed as `twenty`).
- GitNexus groups `backend/frontend/infra` do not contain a `twenty` entry, so no cross-repo fan-out is warranted. The symptom, evidence, and fix are entirely server-side within this single repo.
- Chosen radius: **`twenty` only**.

### A.2 Signal → code
- Signal: `computeWhereConditionParts` / `neq` case → mapped via `Grep` and `mcp__gitnexus__context` to the single definition at `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/api/graphql/graphql-query-runner/utils/compute-where-condition-parts.ts`, lines 20–164.
- Signal: `hasNullEquivalentFieldValue` / `findPostgresDefaultNullEquivalentValue` → defined at `packages/twenty-server/src/engine/api/common/common-args-processors/data-arg-processor/utils/find-postgres-default-null-equivalent-value.util.ts` (imported on line 7).
- Signal: sibling operators `eq`, `is`, `isEmptyArray`, `like`, `ilike` — all in the same switch (lines 56–112), all correctly use `OR ... IS NULL` (additive semantics). Confirms reporter's argument: `neq` is the only case using `OR` in subtractive semantics, which is the defect.
- Only caller: `GraphqlQueryFilterFieldParser.parse` at `packages/twenty-server/src/engine/api/graphql/graphql-query-runner/graphql-query-parsers/graphql-query-filter/graphql-query-filter-field.parser.ts` (two invocations — line 84 for top-level fields and line 148 for composite sub-fields). It passes the produced SQL into TypeORM `queryBuilder.where(sql, params)` / `.andWhere(sql, params)`, so the fragment is combined as a unit — no parenthesization concerns for the `OR` → `AND` swap.

### A.3 Blast radius (`impact` upstream)
- GitNexus reports `impactedCount: 0, risk: LOW` for `computeWhereConditionParts`. (The callgraph walk doesn't surface the one-level caller here — likely because the single caller goes through a method on a class. The real direct caller is the single file above; I've verified manually.)
- Effective blast radius: **one file changes; one caller consumes it unchanged**. The change is purely semantic (correcting SQL), the function signature is untouched, the `params` object shape is unchanged, only the operator token inside the interpolated SQL string flips from `OR` to `AND`.

### A.4 Affected repos
- `ShankHarinath/twenty`

## Phase B — Runtime evidence

Skipped deliberately. This is a pure SQL-construction logic bug reported by inspection of the source, not a production symptom tied to an error signature or crash. There is no structured error to grep in pod logs, and the issue does not cite a deployment context or trace ID. The reporter's reproduction (`filter contacts where firstName neq ''` returns rows with `firstName = ''`) is reproducible in any dev environment that runs Twenty's GraphQL layer; it does not require runtime log correlation. No fabricated signal was manufactured.

## Phase C — Recent commits on the affected file

`git log -- packages/twenty-server/src/engine/api/graphql/graphql-query-runner/utils/compute-where-condition-parts.ts` (most recent 14 commits):

- **`208c0857ee` — `common api - null equivalence (#15926)`, Etienne, 2025-11-24** — *this is the commit that introduced the bug.* The diff adds the null-equivalence branches to every operator. All additive operators (`eq`, `isEmptyArray`, `is`, `like`, `ilike`) correctly use `OR ... IS NULL`. The `neq` branch was written with the same pattern (`OR ... IS NOT NULL`) — that mirror is logically wrong; it should have been `AND ... IS NOT NULL`.
- `67074a75815` — `Harden server-side input validation and auth defaults (#18018)`, 2026-02-18 — rename-only refactor (`uuid` → `paramSuffix`, `Math.random` → `crypto.randomBytes`). Did **not** introduce or touch the boolean operator on line 68. Ruled out as the cause; verified by reading the full diff (attached in investigation trace).
- `9d57bc39e5` — ESLint → OxLint migration, comment-only touch. Irrelevant.
- Other listed commits (searching back to `b792d2a4d37` — file creation on 2024-10-13) do not touch null-equivalence logic.

`git blame -L 66,70` confirms line 68 carries SHA `67074a75815` (the 2026-02 rename), but the logical defect was introduced at `208c0857ee` (2025-11-24). The field of scrutiny here is the `OR` token, not the identifier rename.

## Phase D — Similar past issues and PRs

- `gh issue list --search "neq filter null"` — returns only this issue (#1).
- `gh pr list --search "neq null equivalent"` — no prior or concurrent PR attempting this fix upstream. A Phoenix-tracking branch (`phoenix/1-neq-filter-null-wrong`) was already closed, but no remedial commit has been merged on `develop`.
- The prior PR #15926 (null-equivalence feature itself) contains no explicit test cases for `neq` semantics — the reporter's claim that `neq` exclusion has been silently broken since 2025-11-24 is supported by this absence.

## Phase E — Root-cause hypothesis

**Hypothesis.** In commit `208c0857ee` (PR #15926, "common api - null equivalence"), null-equivalence handling was added to every operator branch in `computeWhereConditionParts`. For additive operators (`eq`, `is`, `like`, `ilike`, `isEmptyArray`), adding `OR fieldReference IS NULL` correctly expands the result set to include rows where the stored value is `NULL` but the user filtered on the null-equivalent literal (e.g., `''`). For the `neq` operator, the author used the same `OR` pattern with `IS NOT NULL` — which is the De-Morgan error the issue describes. The correct negation of `x = v OR x IS NULL` is `x != v AND x IS NOT NULL`, not `x != v OR x IS NOT NULL`. The latter is always-true whenever the column is non-null, effectively neutering every `neq` filter on any TEXT field that has a Postgres default null-equivalent value. Because Twenty's standard non-nullable TEXT columns (firstName, lastName, primaryEmail local/domain parts, primaryPhoneNumber parts, address sub-fields, etc.) store empty strings where users conceptually mean "no value", `hasNullEquivalentFieldValue` is true for nearly every TEXT `neq` call, so the bug manifests on every such filter. The fix is a single-token swap on line 68 of `compute-where-condition-parts.ts`: `OR` → `AND`. No signature change, no caller update, no data migration, no other operator to touch. A focused unit test should assert that `firstName neq ''` excludes rows with `firstName = ''` (and optionally that rows with `firstName = NULL` are also excluded, since `'' IS NOT NULL` is true but `NULL != ''` is `NULL` under SQL three-valued logic — `AND NULL IS NOT NULL` = `AND false` = false, which matches the user's intuitive "exclude all records that look empty" expectation and is symmetric with `eq`).

**Confidence.** 0.95.

- Reporter pinpointed the exact file, line, and logical derivation.
- Source code inspection confirms the described SQL is emitted verbatim from line 68.
- Git archaeology localizes the introduction to PR #15926 (null-equivalence) — matches the semantic category of the defect.
- Sibling operators in the same switch demonstrate the correct additive pattern, strengthening the claim that only `neq` is wrong.
- Fix scope verified: blast radius = one file, one call site, string-only change.
- De Morgan's law derivation is straightforward and matches the reporter's reproduction.

The remaining 0.05 uncertainty reflects the fact that I did not execute the SQL against a running Postgres to observe the produced query plan, and I did not verify there are no *other* code paths (e.g., an alternate filter parser, a ClickHouse runner) that replicate the same pattern and would also need the same fix.

## Open Questions

1. Should the fix also wrap the resulting fragment in parentheses for belt-and-suspenders precedence? Current style across the file does not parenthesize — `AND` binds tighter than `OR` in SQL, so within the fragment the new `field != :v AND field IS NOT NULL` is unambiguous, and TypeORM wraps the whole fragment when combining via `andWhere`. Recommendation: match local style (no extra parens) to keep the diff minimal, but note the decision in the PR body.
2. Is there test coverage for `neq` on null-equivalent fields? I did not find a dedicated spec; the PR should add one (e.g., `compute-where-condition-parts.spec.ts`) asserting that given `hasNullEquivalentFieldValue = true`, `neq` emits `... AND ... IS NOT NULL`.
3. `parse-clickhouse-filter.util.ts` surfaced in the initial query (`buildOperatorCondition`) — does it have the same null-equivalence logic, and if so does it share the defect? I inspected that file's path but not its contents; a quick follow-up should confirm it is either unaffected (no null-equivalence shim) or carries the same bug. This is noted as a secondary item, not a blocker for the primary fix.

## Relevant files (absolute paths)

- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/api/graphql/graphql-query-runner/utils/compute-where-condition-parts.ts` — bug site, line 68.
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/api/graphql/graphql-query-runner/graphql-query-parsers/graphql-query-filter/graphql-query-filter-field.parser.ts` — sole caller (lines 84 and 148).
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/api/common/common-args-processors/data-arg-processor/utils/find-postgres-default-null-equivalent-value.util.ts` — source of `hasNullEquivalentFieldValue` truth value (referenced, not modified).
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/api/clickhouse-query-runners/utils/parse-clickhouse-filter.util.ts` — secondary inspection target (see Open Question 3).

---

```yaml
affected_repos:
  - ShankHarinath/twenty
confidence: 0.95
recommended_next_step: proceed_to_plan
```
