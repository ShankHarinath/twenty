## Triage Report — ShankHarinath/twenty#6

**Classification:** bug  (orchestrator confidence: 0.9)
**Summary:** Concurrent view-group migration runs on v1.18.1 leave `console.time` labels dangling and race on the workspace-migration transaction + post-transaction cache invalidation, causing view-group reorders ("moving stages") to appear frozen until containers are restarted.

### Symptom
- Normalized: Moving (sorting) a kanban stage in the UI intermittently hangs; only a container restart resolves it. Server logs spray `Label '[Runner] ...' already exists for console.time()` and matching `No such label '...' for console.timeEnd()` warnings.
- Error signatures:
  - `Warning: Label '[Runner] Transaction execution' already exists for console.time()`
  - `Warning: Label '[Runner] Cache invalidation flatViewGroupMaps,flatViewMaps' already exists for console.time()`
  - `Warning: Label '[BaseWorkspaceMigrationRunnerActionHandlerService] update_viewGroup executeForMetadata' already exists for console.time()`
  - `Warning: No such label '[Runner] Transaction execution' for console.timeEnd()`
- Reported timeframe: Container version v1.18.1, reported 2026-04-20; reporter ran v1.18.0 for 3 weeks without symptom, so regression landed in the v1.18.0 → v1.18.1 window.

### Runtime signal (from kubectl)
- First-seen: no matching log lines in tail
- Frequency: 0 (no `twenty*` / `crm*` pods on `kind-canonix`)
- Active now: unknown
- Pod/container state: n/a — service not deployed to this cluster
- Target(s): attempted `kubectl --context kind-canonix get pods -A`; cluster hosts `app`, `cortex`, `mcp-gateway`, `mattermost`-less stack; no Twenty deployment present
- Affected entities: none extractable — cannot pull runtime data
- Correlated errors: none available (no pod logs)

### Affected code (from GitNexus)
- **Repos searched:** `twenty` (rationale: issue explicitly names twentycrm container versions and points at `BaseWorkspaceMigrationRunnerActionHandlerService` / `EntityBuilder` / `[Runner]` labels — all live under `packages/twenty-server`; no cross-repo contract involved)
- **Affected repos (fix required in):** `twenty`
- `twenty` · `WorkspaceMigrationRunnerService.run` · `packages/twenty-server/src/engine/workspace-manager/workspace-migration/workspace-migration-runner/services/workspace-migration-runner.service.ts:154-310` — emits `console.time('Runner','Total execution')` at line 165, `'Initial cache retrieval'` at 166, `'Transaction execution'` at 217. On error paths, only `queryRunner.release()` runs in `finally`; **none of the `logger.timeEnd` calls are in `finally`**, so any throw inside the transaction loop leaks a label.
- `twenty` · `WorkspaceMigrationRunnerService.invalidateCache` · same file:108-152 — `logger.time('Runner', 'Cache invalidation ${keys}')` at 115 and matching `timeEnd` at 148 are *not* wrapped in try/finally. After commit `523289efad` this method runs **after** the transaction's try/catch/finally, so a throw inside `flatEntityMapsCacheService.invalidateFlatEntityMaps` or a rejected promise from `Promise.allSettled` → `throw new Error(...)` (line 143-146) leaks the label and exits `run()` without ever closing `'Total execution'` either.
- `twenty` · `BaseWorkspaceMigrationRunnerActionHandlerService.asyncMethodPerformanceMetricWrapper` · `packages/twenty-server/.../interfaces/workspace-migration-runner-action-handler-service.interface.ts:309-325` — `time(...)` / `await method()` / `timeEnd(...)` with **no try/finally**. Any reject from `executeForMetadata`/`executeForWorkspaceSchema` (propagated as `WorkspaceMigrationRunnerException` at 267) skips `timeEnd`. This is exactly the `update_viewGroup executeForMetadata` label seen in the report.
- `twenty` · `LoggerService.time` / `LoggerService.timeEnd` · `packages/twenty-server/src/engine/core-modules/logger/logger.service.ts:61-74` — thin wrappers around the **process-global** `console.time()` / `console.timeEnd()` label registry. The label string `[category] label` is identical across workspaces/invocations (e.g., `[Runner] Total execution`), so two concurrent `run()` calls in the same Node process collide on the same key.
- `twenty` · `UpdateViewGroupActionHandlerService.executeForMetadata` · `packages/twenty-server/.../action-handlers/view-group/services/update-view-group-action-handler.service.ts:46-56` — executes the actual `ViewGroupEntity.update` inside the `queryRunner` transaction. This is the write path for "move a stage".
- `twenty` · `WorkspaceEntityMigrationBuilderService.validateAndBuild` · `packages/twenty-server/.../workspace-migration-builder/services/workspace-entity-migration-builder.service.ts:60-335` — matches the `[EntityBuilder viewGroup] validateAndBuild: 8.06ms` and `entity processing` log lines. Same pattern (multiple `time`/`timeEnd` pairs, no try/finally).
- Blast radius: 4 direct call sites of `WorkspaceMigrationRunnerService.run` — the GraphQL flow that handles view-group ordering (`workspaceMigrationValidateBuildAndRunService.validateBuildAndRunWorkspaceMigration` → `run`), application manifest ingest, and two `1-19` upgrade command runners. The logger helpers (`time`/`timeEnd`) are called from 11 other sites in the server.
- Cross-repo dependencies: none (single-repo fix in `twenty-server`).
- Process participation: none resolved in the GitNexus process graph for `run()` directly; upstream caller `validateBuildAndRunWorkspaceMigration` is the entry point for all metadata mutation (including view-group reorder).
- Misses: none — every pre-extracted signal mapped to `twenty-server` code.

### Historical context
**Similar past issues / PRs:**
- No matching issue in `ShankHarinath/twenty` — only three issues open (#1 neq-filter, #4 AI models, and this #6). `gh search` is disabled by the shim, so upstream `twentyhq/twenty` cannot be queried; confine search radius accordingly.
- Local PR landscape: PRs #2/#3 address issue #1 (neq filter) and PR #5 addresses issue #4 (AI models). No prior work on migration-runner label hygiene or view-group concurrency.

**Recent commits on affected files:**
- `523289efad` 2026-03-25 Paul Rastoin — *"Do not rollback on cache invalidation failure in workspace migration runner (#18947)"* [flag: **primary regression suspect** — moved `invalidateCache` out of the transaction's try/catch into the post-finally section and added its own try/catch that **only logs** on failure; the `this.logger.timeEnd('Runner', 'Total execution')` now lives on the happy path after the catch and is still skipped on upstream throws. This change also slightly expands the window during which `run()` is "in flight" with an open label, making concurrent collisions more visible.]
- `5c745059ad` 2026 — views refactor removing the "core" naming and converter layer [flag: touched adjacent viewGroup code paths; not directly incriminating but increased surface of the update-viewGroup migration path]
- `c107d804d2` — command menu items for workflow triggers [flag: unrelated]

### Root-cause hypothesis
The v1.18.1 regression is in the workspace-migration instrumentation + post-transaction path. Every `console.time` / `console.timeEnd` pair inside `WorkspaceMigrationRunnerService.run`, `WorkspaceMigrationRunnerService.invalidateCache`, `BaseWorkspaceMigrationRunnerActionHandlerService.asyncMethodPerformanceMetricWrapper`, and `WorkspaceEntityMigrationBuilderService.validateAndBuild` is opened outside any `try/finally`, while the categories+labels are process-global identifiers (e.g. `[Runner] Total execution`, `[Runner] Transaction execution`, `[Runner] Cache invalidation flatViewGroupMaps,flatViewMaps`). When a user drags stages quickly, the UI fires multiple overlapping view-group update mutations; because the runner has no per-workspace serialization, two `run()` calls execute concurrently in the same Node process. Node's console-timer bookkeeping is a single-slot map per label — the second `time()` emits "already exists" and does nothing, and the second `timeEnd()` emits "No such label". That alone is noise, but the same concurrency that produces it also causes **real data hazards**: two concurrent transactions on `ViewGroupEntity` update the same rows and (more importantly) commit 523289efad moved `invalidateCache` outside the commit's try/catch, so if the second call's `flushGraphQLOperation(FIND_ALL_VIEWS_GRAPHQL_OPERATION)` or `invalidateFlatEntityMaps` races/throws, the GraphQL cache ends up stale *without rolling back*. The frontend polls the now-stale `FIND_ALL_VIEWS_GRAPHQL_OPERATION` and shows the old order forever — "stuck" until containers restart (process restart wipes the console.time label slot and, more importantly, the cache layer). The console-time warnings are the clearest signal pointing at the culprit code region; the hang is the post-transaction cache-invalidation leak introduced (or made observable) by #18947.

**Confidence:** 0.72

### Evidence chain
1. Log warnings for `[Runner] Transaction execution`, `[Runner] Cache invalidation …`, `[BaseWorkspaceMigrationRunnerActionHandlerService] update_viewGroup executeForMetadata` → these exact label strings appear verbatim in `workspace-migration-runner.service.ts` and `workspace-migration-runner-action-handler-service.interface.ts`; code path confirmed.
2. Every `logger.time(...)` call in the runner, builder, and action-handler wrapper is outside try/finally → any thrown exception leaks the label → subsequent invocations collide → warnings match exactly.
3. `console.time`/`timeEnd` in Node uses a single global label map → concurrent `run()` calls (triggered by a drag-reorder spamming mutations, with no mutex/queue on `WorkspaceMigrationRunnerService`) collide → warnings match.
4. Commit `523289efad` (v1.18.1 window, post-v1.18.0) restructured `run()` so `invalidateCache` runs **after** the transaction try/catch/finally; its inner try/catch merely logs and swallows. A failed `flushGraphQLOperation` therefore leaves the view cache stale while the mutation reports success to the client → UI shows stale stage order → "stuck".
5. Reporter runs v1.18.0 for 3 weeks with no issue; symptom appears on v1.18.1 → regression must be in that diff. The runner service file is the only one in the v1.18.0..HEAD window that touches both the log labels AND the cache-invalidation ordering.
6. No `twenty` pods on `kind-canonix` → could not corroborate runtime trace; confidence held below 0.85.

### Recommended next step
proceed to fix

Fix shape Architect should consider:
- Wrap every `logger.time` / `logger.timeEnd` pair in try/finally so labels cannot leak on throw (runner, action-handler wrapper, entity-migration-builder).
- Make the label strings unique per invocation (include workspaceId + a short nonce) so concurrent runs on the same Node process do not share console.time slots.
- Add per-workspace serialization (Mutex or queue keyed by workspaceId) on `WorkspaceMigrationRunnerService.run` so two view-group reorders on the same workspace cannot race the cache-invalidation path.
- Re-examine #18947: either retain rollback semantics on post-commit cache failure, or make the failure visible to the caller / retry asynchronously so the UI is never served a stale `FIND_ALL_VIEWS_GRAPHQL_OPERATION`.

### Open questions
- Unable to confirm in cluster: `kubectl --context kind-canonix` hosts no Twenty deployment, so frequency/trend of the warnings in the reporter's prod is unverified — advise Architect to request a 2-minute `docker logs` tail from the reporter around a stuck event, or at minimum the log line **immediately preceding** the first "already exists" warning (that preceding throw is the exact leak site).
- Is the reporter's workload multi-tenant on a single Node process? If so, concurrent non-same-workspace mutations also collide and the mutex should be at the **process** level for label uniqueness, and per-workspace only for transactional serialization.
- Upstream (`twentyhq/twenty`) triage/search is blocked by the gh shim — cannot tell whether a fix is already in flight on main. Architect should check before opening a PR.

<!-- MACHINE-READABLE FOOTER — DO NOT REMOVE; downstream skills parse this block -->
<!--phoenix:scout-summary
affected_repos: [twenty]
confidence: 0.72
recommended_next_step: proceed
-->
