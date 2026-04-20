# Triage Report — ShankHarinath/twenty#1

## Summary

**Title:** neq filter returns wrong results on null-equivalent fields
**Classification:** Bug — SQL-generation logic error in the GraphQL query filter compiler.
**Confidence:** 0.97
**Recommended next step:** Apply the one-character fix (`OR` → `AND`) upstream PR #19071 already merged, plus add a unit test.

## Phase A — Symptom → Code

### A.1 Search radius

- `list_repos` shows 11 indexed repos. Home repo is `ShankHarinath/twenty`.
- `group_list` / `group_contracts`: the signal is a self-contained SQL-generation bug inside a single TS file; no contract or cross-service shape is implicated. No grouping needed.
- Chosen radius: **`twenty` only**. Reasoning: the issue body names a specific file (`compute-where-condition-parts.ts`) with a specific operator case; signals resolve fully inside this repo.

### A.2 Queries

- `query("neq operator compute where condition parts hasNullEquivalentFieldValue", repo: twenty)` → resolved `Function:packages/twenty-server/src/engine/api/graphql/graphql-query-runner/utils/compute-where-condition-parts.ts:computeWhereConditionParts` (L19–163).
- `Glob **/compute-where-condition-parts.ts` → single file `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/api/graphql/graphql-query-runner/utils/compute-where-condition-parts.ts`.
- Direct read: the bug is on **line 68** verbatim as quoted in the issue:
  ```
  sql: `${fieldReference} != :${key}${paramSuffix}${hasNullEquivalentFieldValue ? ` OR ${fieldReference} IS NOT NULL` : ''}`
  ```
- Siblings (all correct, additive null handling): `isEmptyArray` L58, `eq` L63, `is` L98, `like` L105, `ilike` L110 — each uses ` OR ... IS NULL` (or equivalent additive form), confirming `neq` is the lone outlier.
- `impact(computeWhereConditionParts, upstream, minConfidence 0.8, repo twenty)` → **LOW risk, 0 impacted symbols.** The function produces a SQL fragment consumed at runtime by TypeORM's `.andWhere(sql, params)` — no static CALLS edges land on it in the graph. Callers discovered by Grep:
  - `packages/twenty-server/src/engine/api/graphql/graphql-query-runner/graphql-query-parsers/graphql-query-filter/graphql-query-filter-field.parser.ts` L84 (scalar field filters) and L148 (composite sub-field filters).
  Both wrap the returned fragment via `queryBuilder.where(sql, params)` / `queryBuilder.andWhere(sql, params)`; TypeORM parenthesizes the fragment before ANDing it with siblings, so `AND ... IS NOT NULL` composes safely without extra grouping.

### A.3 Misses

None. Every signal resolved.

### A.4 Affected repos

**`ShankHarinath/twenty`** — one repo, one file, one line change.

## Phase B — Runtime evidence (kubectl)

**Skipped by design.** This is a pure SQL-generation logic bug; reproduction is deterministic from the source (filter any TEXT column `neq ''`). No pod/log signal is needed to confirm the hypothesis, and the reporter provided none. No cluster evidence was fabricated.

## Phase C — Recent commits on affected files

`git log` on `compute-where-condition-parts.ts` (most recent first):

| SHA | Date | Subject | Relevance |
|---|---|---|---|
| `9d57bc39e5` | — | Migrate from ESLint to OxLint (#18443) | comment-only |
| `67074a75815` | 2026-02-18 | Harden server-side input validation and auth defaults (#18018) | cosmetic: renamed `uuid`→`paramSuffix`; preserved the buggy `OR` |
| `52e57e70fd` | — | Add wildcard documentation for like/ilike/containsIlike filters (#17825) | doc-only |
| **`208c0857ee`** | **2025-11-24** | **common api - null equivalence (#15926)** | **Introduced the bug.** Added `hasNullEquivalentFieldValue` guard to `eq`/`neq`/`is`/`like`/`ilike`/`isEmptyArray`. `neq` was written as the mirror shape of `eq` using `OR` — the logical inverse (`AND`) required for negation was missed. |

`git blame -L 66,70` confirms the `neq` SQL line is owned by `67074a75815` (re-touched during the `crypto.randomBytes` rename), but the `OR` pattern originates in `208c0857ee`.

## Phase D — Similar past issues / PRs

Upstream `twentyhq/twenty` has directly addressed this bug:

- **Issue #19070** (CLOSED, 2026-04-07) — byte-for-byte identical report to ShankHarinath/twenty#1. Closed when fix merged.
- **PR #19071** (MERGED, 2026-03-28) — title: "fix: use AND instead of OR in neq filter for null-equivalent values" by oniani1. Reviewed and approved by Etienne (original author of the null-equivalence work). Diff is a single-character change on line 68:
  ```
  -        sql: `${fieldReference} != :${key}${paramSuffix}${hasNullEquivalentFieldValue ? ` OR ${fieldReference} IS NOT NULL` : ''}`,
  +        sql: `${fieldReference} != :${key}${paramSuffix}${hasNullEquivalentFieldValue ? ` AND ${fieldReference} IS NOT NULL` : ''}`,
  ```
- **PR #19077** (CLOSED) — competing patch that additionally proposed unit tests; superseded by #19071.

This is a fast-path match: the fix pattern is established, reviewed, and already shipped upstream.

## Phase E — Synthesis

### Hypothesis

`computeWhereConditionParts` at `packages/twenty-server/src/engine/api/graphql/graphql-query-runner/utils/compute-where-condition-parts.ts:68` generates `field != :x OR field IS NOT NULL` when the filter value has a Postgres null-equivalent (e.g. `''` for TEXT). For any row where the column is non-null, `IS NOT NULL` is `true` and the `OR` short-circuits the entire predicate to `true`, so null-equivalent rows are never excluded. By De Morgan's law, the correct negation of `eq`'s additive `OR ... IS NULL` is `AND ... IS NOT NULL`. Apply that single substitution.

### Proposed fix

Change line 68:
```diff
- sql: `${fieldReference} != :${key}${paramSuffix}${hasNullEquivalentFieldValue ? ` OR ${fieldReference} IS NOT NULL` : ''}`,
+ sql: `${fieldReference} != :${key}${paramSuffix}${hasNullEquivalentFieldValue ? ` AND ${fieldReference} IS NOT NULL` : ''}`,
```

Optionally (recommended): add a unit test beside this util (no `compute-where-condition-parts.spec.ts` exists today) covering:
1. `neq ''` on a TEXT field — row with `''` must be excluded.
2. `neq ''` on a TEXT field — row with `NULL` must be excluded (consistent with filter semantics).
3. `neq 'Alice'` on a TEXT field — row with `'Alice'` excluded, `'Bob'` included, `NULL` excluded (null-equivalent semantics apply since `''` is the default null-equivalent for TEXT).
4. `neq ''` on a composite sub-field (e.g. `name.firstName`).

### Confidence calibration

**0.97.** Upstream merged an identical one-character fix for the same reported defect. The local tree still contains the exact pre-fix line. Risk on the symbol is LOW. Semantic reasoning (De Morgan) and cross-operator consistency both corroborate.

### Open questions

- None blocking. Minor: the codebase has no spec file for `compute-where-condition-parts.ts`; Architect should decide whether to add one as part of this fix or file a follow-up.

## Files referenced

- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/api/graphql/graphql-query-runner/utils/compute-where-condition-parts.ts` (bug site, line 68)
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/api/graphql/graphql-query-runner/graphql-query-parsers/graphql-query-filter/graphql-query-filter-field.parser.ts` (call sites L84, L148)

## Machine-readable footer

```yaml
affected_repos:
  - ShankHarinath/twenty
confidence: 0.97
recommended_next_step: patch
risk: LOW
primary_file: packages/twenty-server/src/engine/api/graphql/graphql-query-runner/utils/compute-where-condition-parts.ts
primary_symbol: computeWhereConditionParts
primary_line: 68
fix_shape: one_char_substitution
prior_art:
  upstream_issue: twentyhq/twenty#19070
  upstream_pr_merged: twentyhq/twenty#19071
introducing_commit: 208c0857ee
breadcrumb_posted: false
breadcrumb_reason: harness_deny_rule_refused_gh_issue_comment
breadcrumb_file: /tmp/phoenix-issue-comment-1-triage-rev1.md
```
