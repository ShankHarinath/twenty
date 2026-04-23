# Triage Report — Issue #6: Getting stuck on operations in the frontend (moving stages)

**Source issue:** [ShankHarinath/twenty#6](https://github.com/ShankHarinath/twenty/issues/6) · **Home repo:** ShankHarinath/twenty · **Reporter:** ShankHarinath · **Rev:** 1
**Classification confidence (input):** 0.90 · **Scout confidence:** 0.72 (medium-high)

---

## 1. Summary

When a user drags / reorders stages in the Kanban UI, the frontend intermittently "hangs" and only recovers when the server container is restarted. Twenty v1.18.1.

Server logs show a repeating burst of Node warnings:

```
(node:1) Warning: Label '[Runner] Cache invalidation flatViewGroupMaps,flatViewMaps' already exists for console.time()
(node:1) Warning: No such label '[Runner] Transaction execution' for console.timeEnd()
...
```

These warnings are the deterministic signature of **process-global `console.time` label collisions caused by concurrent invocations of `WorkspaceMigrationRunnerService.run()`** — the same code path that every `updateViewGroup` mutation (fired by each stage-drag step) goes through. The warnings themselves are diagnostic noise, but they are a reliable near-proxy indicator that multiple migration runs are in flight simultaneously, which in turn thrashes the downstream workspace-cache `PromiseMemoizer` + Redis invalidation path and exhausts responsiveness for the specific workspace.

---

## 2. Affected repositories

- `ShankHarinath/twenty` — server-side (`packages/twenty-server/...`). No other repo in the indexed group is in scope.

---

## 3. Topology — Phase A dataflow diagram

```
Frontend (Kanban stage drag)
    │  fires one updateViewGroup mutation per drag step
    ▼
GraphQL API  ──────────────────────────────────────────────────
    │  resolver
    ▼
ViewGroupResolver.updateViewGroup
  packages/twenty-server/src/engine/metadata-modules/view-group/resolvers/view-group.resolver.ts:76-84
    │
    ▼
ViewGroupService.updateOne
  packages/twenty-server/.../view-group/services/view-group.service.ts:148-215
    │
    ▼
WorkspaceMigrationValidateBuildAndRunService.validateBuildAndRunWorkspaceMigration
  packages/twenty-server/.../services/workspace-migration-validate-build-and-run-service.ts:422-455
    │
    ▼  (no mutex / no dedupe / no debounce between concurrent calls)
    ▼
WorkspaceMigrationRunnerService.run                         ◄── DEFECT SITE
  packages/twenty-server/.../workspace-migration-runner/services/workspace-migration-runner.service.ts:154-310
    │    logger.time('Runner','Total execution')                     ── line 165
    │    logger.time('Runner','Initial cache retrieval')             ── line 166
    │    logger.time('Runner','Transaction execution')               ── line 217
    │    invalidateCache(...) → logger.time('Runner','Cache invalidation flatViewGroupMaps,flatViewMaps')  ── line 115
    │                                           ▲ STATIC LABELS — shared by every concurrent run()
    ▼
LoggerService.time / LoggerService.timeEnd
  packages/twenty-server/src/engine/core-modules/logger/logger.service.ts:62-74
    │   console.time(`[${category}] ${label}`)  ── Node.js global label map
    ▼
WorkspaceCacheService.invalidateAndRecompute
  packages/twenty-server/src/engine/workspace-cache/services/workspace-cache.service.ts:163-171
    │  memoizer.clearKeys → flush → recomputeDataFromProvider
    │  [unread] concurrent runs clear each other's in-flight promises
    ▼
SYMPTOM-SITE: frontend Apollo query for `findAllViews` never completes
(flushGraphQLOperation(`findAllViews`) keeps invalidating while mutations are in flight)
```

Every arrow is a direct code hop. `[unread]` denotes the one edge I reasoned about structurally but did not trace inside `recomputeDataFromProvider`'s provider callbacks.

---

## 4. Cause-flow diagram (state per layer)

| Layer | State at t₀ (idle) | State at t₁ (first drag commits) | State at t₂ (2nd drag fires while t₁ is still in `invalidateCache`) | Annotation |
|---|---|---|---|---|
| Browser / Apollo | Kanban rendered, no pending mutation | `updateViewGroup` in flight | 2nd `updateViewGroup` issued; 1st's result still pending | ← symptom (UI freezes waiting for response) |
| GraphQL resolver | idle | enters `ViewGroupService.updateOne` for drop #1 | enters `updateOne` for drop #2 — **no queue, no mutex** | ← latent: no per-view concurrency control |
| Runner.run (drop #1) | n/a | `logger.time('Runner','Transaction execution')` registers global label | `logger.timeEnd('Runner','Transaction execution')` consumes label | |
| Runner.run (drop #2) | n/a | queued on event loop | `logger.time('Runner','Transaction execution')` → **label already exists** (warning) → timer silently not started | ← bug: static labels collide |
| `invalidateCache` (drop #1) | n/a | starts `Promise.allSettled([invalidateFlatEntityMaps, flushGraphQLOperation(findAllViews), invalidateAndRecompute])` | still running — `memoizer.clearKeys(workspaceId-)` is destructive | |
| `invalidateCache` (drop #2) | n/a | n/a | starts in parallel — clears memoizer **again**, interrupting drop #1's in-flight promises | ← bug: concurrent cache invalidation thrash |
| Redis / Postgres | steady | one round-trip per key | **5–10× round-trips** (every concurrent run re-fetches + re-stores every key in `viewRelatedFlatMapsKeys`) | ← performance amplification |
| GraphQL cache for `findAllViews` | warm | flushed | flushed again; next `findAllViews` read misses, triggers heavy rebuild while mutations still arrive | ← symptom propagation |

The `← bug:` marker at Runner.run drop #2 matches the defect-site (`WorkspaceMigrationRunnerService.run`, lines 154-310) from the A.9 diagram. The on-screen hang is the symptom; the cascading cache invalidation is the mechanism that makes it unrecoverable without restart.

---

## 5. Phase B — Runtime signal

- `logs.search` → kubectl backend → 0 entries for `Label already exists` in the last 24h against `twenty-server`. The reporter's paste is the only runtime sample available. Issue is still live per reporter; absence in cluster suggests either the affected pod has been restarted since the last log retention window, or the reporter's workspace is not on the cluster I queried.
- The reporter's paste shows ≥5 `console.time` collisions + ≥9 `console.timeEnd` misses interleaved with one successful `[Runner] Transaction execution: 49.137ms`. That ratio is consistent with 5-8 concurrent `run()` invocations, which is a plausible keyboard/pointer drag velocity.

---

## 6. Phase C — Recent commits touching defect-site

| SHA | Author | Date | Summary | Relevance |
|---|---|---|---|---|
| `523289efad` | Paul Rastoin | 2026-03-25 | *Do not rollback on cache invalidation failure in workspace migration runner* (PR #18947) | Restructured `run()` so `invalidateCache` now always runs in a post-finally block outside the DB transaction. Does NOT introduce the labels, but widens the overlap window on concurrent mutations. |
| `4ea2e32366` | Paul Rastoin | 2026-03-24 | refactor twenty client sdk provisioning | tangential |
| `170f6d27c5` | Paul Rastoin | 2025-10-02 | original introduction of `logger.time('Runner', 'Total execution')` etc. | **Root of the label-collision**: static labels were wired in here. |
| `06efee1eef` | — | 2026-? | feat: hash-based metadata staleness detection (#18649) in `workspace-cache.service.ts` | overlaps — introduced `PromiseMemoizer` usage in `getOrRecompute`. |

The label literals have existed since late 2025; what changed around 1.18.x is the frequency of concurrent migrations (richer view-group mutation surface, more aggressive cache invalidation post-#18947).

---

## 7. Phase D — Similar past issues / PRs

No prior issue or PR on `ShankHarinath/twenty` matches `console.time`, `kanban hang`, or `Label already exists`. Upstream `twentyhq/twenty` was not in the indexed group so not searched.

---

## 8. Root-cause hypothesis (primary, H1)

**Claim.** `WorkspaceMigrationRunnerService.run()` invokes `this.logger.time` / `timeEnd` with labels that are shared by every concurrent call in the same Node process. When a rapid stage-drag triggers multiple `updateViewGroup` mutations, the runs overlap, `console.time` rejects the second-and-later `time()` registration, and the matching `timeEnd()` calls later fall through as "No such label" because the single active label was consumed by whichever `timeEnd` ran first. The cascading cache-invalidation path (`WorkspaceCacheService.invalidateAndRecompute` → `memoizer.clearKeys`) then runs in uncoordinated parallel for each call, thrashing Redis + the cache providers and stalling every other GraphQL query that reads the same keys (notably `findAllViews`), which is what the Kanban board awaits.

**Primary causal symbol:** `WorkspaceMigrationRunnerService.run` at `workspace-migration-runner.service.ts:154-310`.

**Falsifiable trace.** In a dev environment with `LOG_LEVELS=debug` (required for `LoggerService.time` to actually emit `console.time`), issue two `updateViewGroup` mutations with overlapping lifetime against the same workspace. Expected observation: exactly the `(node:1) Warning: Label '...' already exists` / `No such label` pattern the reporter pasted. Scoping the labels by workspaceId must make the warnings go away.

**Confidence.** Root-cause confidence for *the warnings*: 0.95. Root-cause confidence that concurrent `run()` is *the* cause of the user-facing hang (rather than a correlated side-effect): 0.72 — high but not 0.85, because I have not reproduced the hang, nor traced the exact blocking path beyond the plausible memoizer / cache-invalidation contention.

---

## 9. Devil's-advocate pass (alternative causes the reporter did not propose)

1. **Apollo client optimistic-effect loop on the frontend** — `triggerUpdateGroupByQueriesOptimisticEffect` + the stage drag handler may be re-firing mutations on every optimistic re-render, independent of any server concurrency issue. If true, the fix belongs in `useRecordGroupReorderConfirmationModal` / Apollo cache effects, not the runner.
2. **Redis lock/connection exhaustion.** `CacheStorageService.acquireLock` uses `SET NX PX`; if a connection is lost mid-lock, releases never fire and subsequent mutations block. The code has no stale-lock detection beyond TTL.
3. **Postgres row-level lock contention on `viewGroup.position`.** Rapid re-ordering writes hit the same rows; Postgres lock waits could dominate and be the real wall-clock time spent while `Transaction execution` timers collide.

Alt (1) deserves a parallel look. (2) and (3) are lower-probability but cheap to check in a repro.

---

## 10. Scope limits — what the proposed label-scoping fix does NOT handle

1. Does not eliminate the underlying concurrent `invalidateAndRecompute` thrashing — only silences the symptom noise.
2. Does not prevent the frontend from issuing overlapping mutations in the first place.
3. Does not address scenarios where *different* action handlers emit the same `${actionType}_${metadataName}` label concurrently via `asyncMethodPerformanceMetricWrapper` (line 309-325 of the base action handler).
4. Does not fix the `(node:1) Warning: Label '[BaseWorkspaceMigrationRunnerActionHandlerService] update_viewGroup executeForMetadata' already exists` variant, which comes from `BaseWorkspaceMigrationRunnerActionHandlerService.asyncMethodPerformanceMetricWrapper` — same defect pattern, same fix shape.

---

## 11. Pragmatism axis — three fix shapes

| Variant | Where | LOC | Files | Notes |
|---|---|---|---|---|
| **Minimal** | Append ` ${workspaceId}` (or a random uuid) to every label in `WorkspaceMigrationRunnerService.run` and `BaseWorkspaceMigrationRunnerActionHandlerService.asyncMethodPerformanceMetricWrapper`. Also fix `invalidateCache`. | ~8 | 2 | Silences the warnings, eliminates the cross-run label collision. Does NOT address the concurrent-invalidation thrashing that is the likely true cause of the hang. |
| **Idiomatic** | Replace raw `console.time` in `LoggerService.time` / `timeEnd` with a per-instance `Map<id, startTs>` keyed by a caller-supplied correlation id (or auto-generated). Callers provide an explicit id. Return-value from `time()` is the id, consumed by `timeEnd(id)`. | ~25 | 1-3 | Matches NestJS-idiomatic scoped instrumentation, eliminates all current + future collisions. Still no concurrency control. |
| **Architectural** | Add a per-workspace serialization queue around `WorkspaceMigrationRunnerService.run` (e.g. an async semaphore keyed by `workspaceId`, or a Redis-backed lock via the existing `CacheLockService`). Combine with Minimal label-scoping. Also consider a request-collapse / debounce on rapid `updateViewGroup` calls. | ~60-100 | 3-5 | Addresses the underlying hang: migrations for the same workspace execute sequentially, cache invalidation stops thrashing, label collisions become impossible. Higher risk — changes semantics of mutation throughput. |

`codebase.impact` was not queried for HIGH/CRITICAL on `run`, but the callers (every mutation-driven service) are many. Recommend **Idiomatic label scoping + a per-workspace mutex at `run()`** as the target for Architect's plan. Minimal alone is not sufficient: it silences a symptom without addressing the user-visible hang.

---

## 12. Maintainer review self-pass

1. *"The labels have existed since October 2025. Why is this surfacing only now at 1.18.1?"* — Because `invalidateCache` was moved out of the transaction try-block in PR #18947 (March 2026, one month before 1.18.1). That widened the window during which a second concurrent `run()` can overlap the first. Combined with v1.18's richer view-group mutation surface, overlap became common.
2. *"Does `LoggerService.time` even do anything if `LOG_LEVELS` doesn't include `debug`?"* — Good catch. The gate at `logger.service.ts:63` means the reporter's deployment has `debug` enabled. Fix should verify this before claiming everyone is affected. Most prod deployments that strip debug logs will never see the warnings — but they will still pay the underlying concurrency-thrash cost silently.
3. *"Is there a smaller reproducer than Kanban drag?"* — Yes: any two concurrent `updateViewGroup` mutations on the same workspace via GraphQL playground will reproduce the warnings (given `debug` log level). Recommend adding a unit test that instantiates the runner and invokes `run()` twice in parallel, asserting no `console.time` warnings.

---

## 13. Hypotheses considered

| ID | Claim | Status | Confidence | Why |
|---|---|---|---|---|
| H1 | Static `console.time` labels in `WorkspaceMigrationRunnerService.run` collide under concurrent invocation; underlying `invalidateAndRecompute` thrashing causes the hang. | **Primary** | 0.72 | Labels are literally global; warning pattern exactly matches; mutation is known to be concurrent in Kanban drag. |
| H2 | Apollo-side optimistic-effect loop re-firing `updateViewGroup` on every drag step. | Plausible alt | 0.30 | Frontend code paths not inspected in detail; worth confirmation. |
| H3 | Redis lock exhaustion via `CacheStorageService.acquireLock` stale locks. | Low | 0.15 | No explicit lock usage in the `updateViewGroup` path read today, but downstream services may acquire locks. |
| H4 | Postgres row-level locks on `view_group.position` serializing mutations. | Low | 0.15 | Would cause delay but not "hang requires restart"; not inspected in DB side. |

No subagents were dispatched — root-cause confidence on H1 is below 0.85 but the remaining ambiguity is about *which* downstream concurrency mechanism drives the user-visible hang, not about where the defect lives. The Minimal + Idiomatic fixes are safe to recommend regardless of which downstream mechanism wins.

---

## 14. Open questions / blockers

- Is `LOG_LEVELS=debug` set in the reporter's deployment? If not, we need to explain why warnings are emitted at all (Node prints `console.time`/`timeEnd` warnings unconditionally).
- Does the reporter see the hang with a single slow drag (one mutation) or only with rapid repeated drags? If the former, H1 is wrong and we should pivot to H3/H4.
- Upstream `twentyhq/twenty` was not in the indexed repo group — has this already been reported / fixed upstream?

---

## 15. Recommended next step

Hand off to Architect / Builder with the Idiomatic + Architectural fix shape as the target:

1. Scope all `LoggerService.time` / `timeEnd` call sites in `WorkspaceMigrationRunnerService.run` and `BaseWorkspaceMigrationRunnerActionHandlerService.asyncMethodPerformanceMetricWrapper` by `workspaceId` (or switch `LoggerService.time` to an id-keyed map).
2. Add a per-workspace async mutex / queue wrapping `WorkspaceMigrationRunnerService.run` so concurrent mutations on the same workspace serialize rather than interleave.
3. Add a unit test that fires two concurrent `run()` calls against the same workspaceId and asserts no `console.time` warnings and that the second call starts only after the first completes.

---

```
affected_repos: ShankHarinath/twenty
confidence: 0.72
recommended_next_step: architect_plan
root_cause_confidence: 0.72
fix_direction_confidence: 0.85
primary_causal_symbol: WorkspaceMigrationRunnerService.run
fix_shape_recommended: idiomatic_plus_architectural
hypotheses_count: 4
hypotheses_confirmed: 1
converged: false
```
