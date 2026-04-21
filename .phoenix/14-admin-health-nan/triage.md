# Triage Report — Issue #14

**Repo (home):** ShankHarinath/twenty
**Issue:** #14 — Bug: NaN displayed in admin health indicators for Redis hit rate and DB cache ratio
**Reporter:** ShankHarinath
**Rev:** 1
**Triaged:** 2026-04-21

---

## Summary

Two independent but co-located defects in the admin-panel health indicators cause `NaN%` to render in the UI:

1. **Redis `hitRate`** — guard uses the truthy string `statsData.keyspace_hits`, which is `"0"` (truthy) on a fresh Redis. The branch then evaluates `0 / (0 + 0) = NaN`, stringified as `"NaN%"`.
2. **Database `cacheHitRatio`** — the SQL query `SELECT sum(heap_blks_hit) * 100.0 / (sum(heap_blks_hit) + sum(heap_blks_read)) as ratio FROM pg_statio_user_tables` returns a single row with `ratio = NULL` when `pg_statio_user_tables` is empty (and also divides by zero when both sums are 0). `parseFloat(null)` → `NaN`, `Math.round(NaN)` → `NaN`, concatenated with `'%'`.

Both are straightforward numeric-guard bugs. No cross-service coupling; impact analysis shows LOW risk with just 2 direct importers (the admin-panel module wiring + `AdminPanelHealthService`). Confidence is very high — the code paths exactly match the reporter's description, reproduced by reading the files.

## Phase A — Code mapping

### A.1 Search radius
- Issue filed against `ShankHarinath/twenty`. Filenames (`redis.health.ts`, `database.health.ts`) are Twenty-specific Nest server files. Not a cross-service bug. Groups (`backend`, `frontend`, `infra`) do not list Twenty, so no group_query needed.
- **Scope limited to `twenty` repo.**

### A.2 Signals mapped to code

**Redis `hitRate` bug** — `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/core-modules/admin-panel/indicators/redis.health.ts` lines 73–80:

```ts
hitRate: statsData.keyspace_hits
  ? Math.round(
      (parseInt(statsData.keyspace_hits) /
        (parseInt(statsData.keyspace_hits) +
          parseInt(statsData.keyspace_misses))) *
        100,
    ) + '%'
  : '0%',
```

`statsData` is produced by `parseInfo()` (line 35–47) which assigns raw string values. For a fresh Redis, `INFO stats` returns `keyspace_hits:0\r\nkeyspace_misses:0`. The string `"0"` is truthy, so the ternary takes the first branch; `0 / 0 = NaN`, then `Math.round(NaN) + '%'` → `"NaN%"`.

Fix pattern: gate on a numeric-total check, e.g.
```ts
const hits = parseInt(statsData.keyspace_hits ?? '0', 10);
const misses = parseInt(statsData.keyspace_misses ?? '0', 10);
const total = hits + misses;
hitRate: total > 0 ? Math.round((hits / total) * 100) + '%' : '0%',
```

**Database `cacheHitRatio` bug** — `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/core-modules/admin-panel/indicators/database.health.ts` line 57–61 (query) and line 92 (formatting):

```ts
this.dataSource.query(`
  SELECT
    sum(heap_blks_hit) * 100.0 / (sum(heap_blks_hit) + sum(heap_blks_read)) as ratio
  FROM pg_statio_user_tables
`),
// ...
cacheHitRatio: Math.round(parseFloat(cacheHitRatio.ratio)) + '%',
```

`pg_statio_user_tables` has zero rows on a fresh database (no user tables yet) → `sum()` returns `NULL` for each → row returned is `{ ratio: null }`. `parseFloat(null)` returns `NaN`; `Math.round(NaN)` is `NaN`; concatenation produces `"NaN%"`. There's a secondary division-by-zero case if user tables exist but have had no block activity.

Fix pattern: use SQL `COALESCE` + `NULLIF` guarding, or a JS-side guard:
```ts
// SQL-side fix
SELECT COALESCE(
  sum(heap_blks_hit) * 100.0
    / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0),
  0
) as ratio
FROM pg_statio_user_tables
```
plus a JS fallback `Number.isFinite(parsed) ? Math.round(parsed) + '%' : '0%'`.

### A.3 Blast radius (impact analysis)

Ran `impact(RedisHealthIndicator, upstream)` and `impact(DatabaseHealthIndicator, upstream)` — both **LOW risk**:
- **d=1 (WILL BREAK):** `admin-panel.module.ts`, `admin-panel-health.service.ts` — these only import the class; fix will not change the public API (same `isHealthy()` signature, same `details` shape). No caller updates needed.
- **d=2 (LIKELY AFFECTED):** `admin-panel.resolver.ts`, `client-config.module.ts`, `core-engine.module.ts` — transitive wiring only.

Consumer (`AdminPanelHealthService.getSystemHealthStatus` / `getIndicatorHealthStatus`, `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/core-modules/admin-panel/admin-panel-health.service.ts`) forwards `details` as an opaque JSON blob — purely formatting-safe changes.

### A.4 Affected repos

Single repo: **twenty**. Both files live in `packages/twenty-server`; fix is one PR touching two `.ts` files plus their unit tests.

## Phase B — Runtime evidence (kubectl)

Skipped deliberately. The defect manifests as an incorrect string in the GraphQL response consumed by the admin UI — it is not surfaced as an error log or exception from the Nest server, so pod logs cannot corroborate or refute it. The reporter's reproduction steps (fresh Redis / empty pg_statio_user_tables → visible `NaN%`) are deterministic from the source alone. If later asked to confirm against a cluster, I'd run `kubectl --context kind-canonix logs deploy/twenty-server --tail=500 --timestamps | grep -iE 'keyspace|cacheHit'` but would not expect hits.

## Phase C — Recent commits on affected files

`git log --since="18 months ago" --oneline` on both files returned exactly one touch:

- `37b9a55382` — "Migrate metrics to prometheus (#17810)" (Charles Bochet, 2026-02-09). Path reorganization of `admin-panel`; the arithmetic at lines 73–80 (redis) and line 92 (database) was not changed, only moved. **This commit did not introduce the bug.**

Conclusion: the bugs predate the Prometheus migration and have been latent since the indicators were originally authored. This matches the reporter's characterization as a "long-standing design flaw."

## Phase D — Similar past issues / PRs

- `gh issue list --search "NaN health"` → only Issue #14 itself.
- `gh pr list --search "NaN health indicator hitRate cacheHitRatio"` → no hits.

No prior fix pattern inside the repo to crib from, but the remedy is canonical (guarded-division + `COALESCE/NULLIF` at the SQL boundary).

### Existing tests

Unit tests exist and use mocks, but neither covers the zero/empty edge case:
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/core-modules/admin-panel/__tests__/redis.health.spec.ts` — "returns up status" mock uses `keyspace_hits:90\r\nkeyspace_misses:10` (non-zero). No zero-activity test.
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/core-modules/admin-panel/__tests__/database.health.spec.ts` — mock returns `[{ ratio: '95.5' }]`. No null-ratio test.

Fix must add regression tests for both edge cases.

## Phase E — Root cause hypothesis

**Hypothesis:** The two `NaN%` displays are independent bugs rooted in the same class of mistake — arithmetic on values that can legitimately be zero/null at rest, without guarding the denominator or handling `NULL` at the SQL boundary. For Redis, the ternary's truthy-string guard is semantically wrong (string `"0"` is truthy in JS); for Postgres, the aggregate query returns a single row with a `NULL` ratio whenever the source view is empty, and `parseFloat(null) → NaN` is never caught. Fixing both is a small, local change in two files with unit-test additions; LOW blast radius; no behavior change for the healthy path.

**Confidence:** 0.95 — both bugs are reproducible purely from reading the source code, the reporter's description precisely matches the code paths, the fix is canonical, and impact analysis confirms no downstream breakage.

## Open questions

- Should the "undefined" state be represented as `'0%'` (today's fallback for Redis) or `'N/A'` (reporter's suggestion)? These are observationally different: `0%` implies "we measured zero cache efficiency," while `N/A` implies "not yet measurable." Recommend `'N/A'` for both indicators when total traffic is zero, to avoid misleading operators into thinking the cache is failing. Flagging for Architect to decide.
- No `.gitignore`/`.phoenix/config.yaml` files inspected for post-fix CI gates; unit tests via `jest` should be sufficient.

## Recommended next step

Proceed to plan/build. Scope is tightly contained in two files in `twenty` repo; one PR, two files modified, two test files updated. No cross-repo coordination needed.

---

```yaml
# machine-readable footer
issue: 14
home_repo: ShankHarinath/twenty
rev: 1
affected_repos:
  - twenty
confidence: 0.95
risk: LOW
recommended_next_step: plan
files_to_edit:
  - packages/twenty-server/src/engine/core-modules/admin-panel/indicators/redis.health.ts
  - packages/twenty-server/src/engine/core-modules/admin-panel/indicators/database.health.ts
tests_to_update:
  - packages/twenty-server/src/engine/core-modules/admin-panel/__tests__/redis.health.spec.ts
  - packages/twenty-server/src/engine/core-modules/admin-panel/__tests__/database.health.spec.ts
```
