## Triage Report — ShankHarinath/twenty#4

**Classification:** bug  (orchestrator confidence: 0.95)
**Summary:** AI provider API keys saved through the admin panel update the DB-backed config but never trigger a reload of the in-memory `AiModelRegistryService`, so models stay unregistered until the next full server restart.

### Symptom

- Normalized: On self-hosted Twenty, a freshly-saved AI provider API key (Mistral in the repro; applies to any catalog provider) is accepted by the admin panel but subsequent AI usage fails with "No AI models are available" until the server is restarted.
- Error signatures: `No AI models are available. Configure at least one AI provider.`
- Reported timeframe: unspecified — user-driven, reproducible on demand.
- Affected entities: self-hosted deployments where `IS_CONFIG_VARIABLES_IN_DB_ENABLED=true` (required to even use the admin-panel write path) and provider keys were NOT present in `process.env` at boot.

### Runtime signal (from logs.search)

- First-seen: not queried — reporter did not supply a service name, trace id, or timestamp; the symptom is deterministic from the code path and running `logs.search` without a target would be noise.
- Frequency: n/a.
- Active now: unknown.
- Service state: n/a.
- Target(s): none queried.
- Backend source: n/a.
- Correlated errors: n/a.
- Trace excerpts: n/a.

`logs.search` intentionally skipped; the defect is confirmed by static tracing of the admin-panel write path vs. the registry build path.

### Dataflow diagram

```
User enters Mistral key in Admin → AI → Models
  │
  ▼
useConfigVariableActions.handleUpdateVariable         [front]      twenty-front/src/modules/settings/admin-panel/config-variables/hooks/useConfigVariableActions.ts:30
  │  GraphQL mutation updateDatabaseConfigVariable (key="MISTRAL_API_KEY")
  ▼
AdminPanelResolver.updateDatabaseConfigVariable       [resolver]   twenty-server/src/engine/core-modules/admin-panel/admin-panel.resolver.ts:277
  │  call
  ▼
TwentyConfigService.update                            [service]    twenty-server/src/engine/core-modules/twenty-config/twenty-config.service.ts:83
  │  call
  ▼
DatabaseConfigDriver.update → writes DB, updates cache [driver]    twenty-server/src/engine/core-modules/twenty-config/drivers/database-config.driver.ts:71-83
  │  ── NO CALL TO aiModelRegistryService.refreshRegistry() ──                                       ◀── defect site
  ▼
(later) AgentChatResolver.sendChatMessage             [resolver]   twenty-server/src/engine/metadata-modules/ai/ai-chat/resolvers/agent-chat.resolver.ts:120
  │  call
  ▼
AiModelRegistryService.getAvailableModels()           [service]    twenty-server/src/engine/metadata-modules/ai/ai-models/services/ai-model-registry.service.ts:152
  │  returns [] because registry was built at boot, before key existed
  ▼
throw AgentException("No AI models are available.")                                                  ◀── symptom
```

Every arrow is a code-level hop. The defect is the missing edge, not a wrong edge — `updateDatabaseConfigVariable` MUST call `refreshRegistry()` the way `addAiProvider` / `removeAiProvider` / `addModelToProvider` / `removeModelFromProvider` already do (admin-panel.resolver.ts:406, 423, 478, 511).

### Cause-flow diagram

```
reported symptom: "AI chat returns 'No AI models are available' after saving an API key in the admin panel"

layer                         state at this layer                                             observation
─────                         ────────────────────                                           ──────────────
Admin UI                      key typed, save button clicked                                  correct
GraphQL updateDatabaseConfigVariable   payload reaches server                                 correct
TwentyConfigService.update    DB row upserted, in-memory cache updated                        correct
ProviderConfigService.getResolvedProviders()   WOULD now resolve {{MISTRAL_API_KEY}} to the saved value   correct (if called)
AiModelRegistryService.modelRegistry   still reflects boot-time snapshot with zero providers  ← bug: no refreshRegistry() call on this write path
AgentChatResolver.sendChatMessage   sees getAvailableModels().length === 0                    ← symptom
```

**Reconciliation:** the defect site in the dataflow diagram (missing `refreshRegistry()` call after `twentyConfigService.update`) lines up with the `← bug:` marker on the registry layer in the cause-flow diagram. Consistent.

### Affected code (from codebase.*)

- **Repos searched:** twenty (rationale: reporter explicit, sole home repo; no cross-group dependency implicated).
- **Affected repos (fix required in):** twenty.
- `twenty` · `AdminPanelResolver.updateDatabaseConfigVariable` · `packages/twenty-server/src/engine/core-modules/admin-panel/admin-panel.resolver.ts:277-285` — writes config but does not refresh the AI registry.
- `twenty` · `AdminPanelResolver.createDatabaseConfigVariable` · `packages/twenty-server/src/engine/core-modules/admin-panel/admin-panel.resolver.ts:265-273` — same gap; first-time-save path for a catalog key that was never set before.
- `twenty` · `AdminPanelResolver.deleteDatabaseConfigVariable` · `packages/twenty-server/src/engine/core-modules/admin-panel/admin-panel.resolver.ts:287-295` — symmetrical gap (removing a key should also force re-registration so the provider goes away).
- `twenty` · `AiModelRegistryService.buildModelRegistry` · `packages/twenty-server/src/engine/metadata-modules/ai/ai-models/services/ai-model-registry.service.ts:62-71` — built exactly once in the constructor; only `refreshRegistry()` re-runs it.
- `twenty` · `ProviderConfigService.resolveTemplate` · `packages/twenty-server/src/engine/metadata-modules/ai/ai-models/services/provider-config.service.ts:47-74` — correctly prefers DB-backed config over `process.env`, so the data would be available on the next `getResolvedProviders()` — the registry just never re-reads.
- `twenty` · `isProviderConfigured` · `packages/twenty-server/src/engine/metadata-modules/ai/ai-models/utils/is-provider-configured.util.ts:3` — registry uses this to decide whether to instantiate the SDK. At boot time with no key, it returns false and the provider is skipped.
- `twenty` · `DatabaseConfigDriver.refreshAllCache` · `packages/twenty-server/src/engine/core-modules/twenty-config/drivers/database-config.driver.ts:124-147` — a periodic cron exists for config variables, but it only refreshes the config cache; it does NOT call `AiModelRegistryService.refreshRegistry()`, so even the scheduled path doesn't recover from this bug.
- Blast radius (codebase.impact depth 2 on `refreshRegistry`): 1 direct callee (`buildModelRegistry`), 2 transitive; risk LOW. Callers today: only the four custom-provider admin mutations, which is precisely the asymmetry that causes the bug.
- Cross-repo dependencies: none. Twenty is the only surface.
- Process participation: none detected beyond the "AI chat send" flow at agent-chat.resolver.ts:120.
- Sibling scan hits:
  - `packages/twenty-server/src/engine/metadata-modules/ai/ai-generate-text/controllers/ai-generate-text.controller.ts:36` — same error message path; same underlying root cause (registry empty).
  - `packages/twenty-server/src/engine/metadata-modules/ai/ai-models/constants/ai-models-types.const.spec.ts:132` — existing test asserts the error string; useful regression target.
  - `packages/twenty-apps/examples/postcard/src/components/generate-post-card-component-effect.tsx:60` — example app surfaces the same error to UI users; not a bug, but confirms the string contract.
  - `packages/twenty-server/src/engine/metadata-modules/ai/ai-models/ai-providers.json` — catalog with `{{MISTRAL_API_KEY}}` templates that require runtime resolution every build.
- Misses: none material. "process.env" in the report maps to the template-resolution fallback in `ProviderConfigService.resolveTemplate` (provider-config.service.ts:73), which explains exactly why the env-var workaround works.

### Historical context

**Similar past issues / PRs:**
- `issues.search` and `vcs.pr_search` for `ShankHarinath/twenty` returned zero hits for prior "No AI models available" / registry-refresh issues — this is the first report of the specific gap on the DB-config write path.

**Recent commits on affected files** (from `git log`):
- `908aefe7c1` 2026-03-21 Félix Malfait — "feat: replace hardcoded AI model constants with JSON seed catalog (#18818)" [flag: this PR introduced the JSON-catalog + admin-panel add/remove flow and added `refreshRegistry()` calls on `addAiProvider` / `removeAiProvider`; the sibling `createDatabaseConfigVariable` / `updateDatabaseConfigVariable` / `deleteDatabaseConfigVariable` pathways were NOT wired to the same refresh. High likelihood this is the regression point.]
- `7a341c6475` 2026-04 "feat: support authType on AI providers for IAM role authentication (#19016)" — added another configured-check axis (`authType`) but did not revisit the config-variable write path; the same asymmetry survives.
- `9a3852bf04` "feat: add two-layer AI model availability filtering (#18170)" — unrelated filter layer; not causal.

**What else did #18818's author change (same PR):** the PR also introduced `isProviderConfigured`, `buildCompositeModelId`, and the entire admin-panel "add custom provider" UX. Those paths correctly call `refreshRegistry()`. The omission is specifically on the config-variable CRUD resolvers, which pre-existed and were not audited for AI-registry coupling.

### Hypotheses considered

Scout explored the following hypotheses in Phase E1. The confidence gate fired at E1 (top hypothesis ≥ 0.85 with all signals mapped and static-trace corroboration), so the multi-investigator Phase E2 was skipped.

- **H1 — Config-variable CRUD mutations do not refresh the AI model registry, so API keys saved via the admin panel are invisible to the in-memory registry until restart.** · subagent: dataflow_tracer (not dispatched) · result: confirmed (static) · final_conf: 0.92
  - Falsifiable trace outcome: `git grep refreshRegistry` shows calls only from `addAiProvider` / `removeAiProvider` / `addModelToProvider` / `removeModelFromProvider` (all at admin-panel.resolver.ts:406/423/478/511). Zero calls from `createDatabaseConfigVariable` / `updateDatabaseConfigVariable` / `deleteDatabaseConfigVariable`. Predicate confirmed.
  - Key evidence: `packages/twenty-server/src/engine/core-modules/admin-panel/admin-panel.resolver.ts:265-295` (the three gap sites) and `.ts:62-71` of the registry service (registry built once in constructor).
- **H2 — `isProviderConfigured` incorrectly returns false for Mistral when key is saved (semantic defect in the predicate).** · subagent: library_contract_checker · result: refuted · final_conf: 0.05
  - Falsifiable trace outcome: `isProviderConfigured = (c) => !!(c.apiKey || c.accessKeyId || c.authType)` at `is-provider-configured.util.ts:3` — once the registry re-reads, `c.apiKey` resolves to the saved string and the predicate returns true. Refuted.
- **H3 — `ProviderConfigService.resolveTemplate` short-circuits to stale `process.env` and never consults the DB.** · subagent: dataflow_tracer · result: refuted · final_conf: 0.05
  - Falsifiable trace outcome: `provider-config.service.ts:61-73` explicitly tries the DB-backed `TwentyConfigService.get` FIRST and only falls back to `process.env` on miss. So the resolution order is correct; the problem is that no one triggers re-resolution. Refuted.
- **H4 — Multi-node deployment staleness: one pod's in-memory registry doesn't observe another pod's config writes.** · subagent: git_archaeologist · result: inconclusive · final_conf: 0.25
  - Falsifiable trace outcome: true for multi-pod deployments (no cross-node invalidation broadcast) but NOT the primary cause — the repro here is single-node (Railway) and fails on the very pod that wrote the config. H1 explains the single-node case completely. Note for scope-limits.

**Consolidator synthesis:** `primary_causal_symbol` = `twenty-server/src/engine/core-modules/admin-panel/admin-panel.resolver.ts:277` (and its sibling `:265` / `:287`); 1 confirmed, 2 refuted, 1 inconclusive (orthogonal multi-pod concern). Merges: none. Contradictions: none. Converged: true.

*Confidence gate fired — H1 had root_cause_confidence ≥ 0.85 with all signals mapped and static-trace corroboration; multi-hypothesis investigator dispatch was skipped.*

### Root-cause hypothesis

The `AiModelRegistryService` builds its in-memory registry exactly once — in its constructor at server boot — and is refreshed **only** when the custom-provider admin mutations (`addAiProvider` / `removeAiProvider` / `addModelToProvider` / `removeModelFromProvider`) explicitly call `refreshRegistry()` after persisting changes. The parallel and equally legitimate write path — the config-variable CRUD mutations `createDatabaseConfigVariable` / `updateDatabaseConfigVariable` / `deleteDatabaseConfigVariable` in `admin-panel.resolver.ts:265-295` — persists the API-key value into the DB-backed config cache but does not trigger the same refresh. Because catalog providers (Mistral, OpenAI, Anthropic, Google, xAI) are configured via config variables of the form `{{MISTRAL_API_KEY}}` that are resolved at `buildModelRegistry` time, and because the very first boot of a self-hosted instance has no such keys set, `isProviderConfigured` returns false for every catalog provider, the SDK instance is never created, and the registry map is empty. Saving the key later through the admin panel correctly updates the DB and the config cache, but the registry — still holding the empty boot-time snapshot — continues to return zero available models. The user sees "No AI models are available" until the server process is restarted. The environment-variable workaround bypasses the bug because `process.env.MISTRAL_API_KEY` is already present at boot time, so the registry is populated in the constructor and no subsequent refresh is needed.

**Root-cause confidence:** 0.92
**Fix-direction confidence:** 0.90

### Evidence chain

1. `updateDatabaseConfigVariable` resolver writes config but never touches the AI registry → bug site. `admin-panel.resolver.ts:277-285`.
2. `createDatabaseConfigVariable` has the identical gap (first-save case) → same bug, different entry. `admin-panel.resolver.ts:265-273`.
3. `deleteDatabaseConfigVariable` has the symmetric gap (removing a key should invalidate the SDK instance). `admin-panel.resolver.ts:287-295`.
4. The four custom-provider mutations DO call `refreshRegistry()` — proves the authors knew about the coupling and had a template; the config-variable CRUD just wasn't updated to match. `admin-panel.resolver.ts:406, 423, 478, 511`.
5. `AiModelRegistryService.buildModelRegistry` is called only from the constructor and `refreshRegistry()` — no other entry. `ai-model-registry.service.ts:54-60, 359-361`.
6. `ProviderConfigService.resolveTemplate` prefers DB-backed config over `process.env` — so the saved value WOULD be visible to the registry if only the registry re-read it. `provider-config.service.ts:47-74`.
7. `isProviderConfigured` skips SDK instantiation when `apiKey`, `accessKeyId`, and `authType` are all falsy — at boot with no env keys, every catalog provider is skipped. `is-provider-configured.util.ts:3` and `ai-model-registry.service.ts:84-114`.
8. `AgentChatResolver.sendChatMessage` is the user-facing symptom emitter: when `getAvailableModels().length === 0`, it throws the exact error the reporter saw. `agent-chat.resolver.ts:120-138`.
9. `DatabaseConfigDriver.refreshAllCache` runs periodically and refreshes the config cache — but it does NOT call `AiModelRegistryService.refreshRegistry()`, so even waiting does not recover. `database-config.driver.ts:124-147`.
10. Regression window: the JSON-catalog refactor `908aefe7c1` (#18818, 2026-03-21) introduced the custom-provider mutations with their `refreshRegistry()` calls; the pre-existing config-variable CRUD resolvers were not updated to match. `git log --follow packages/twenty-server/src/engine/metadata-modules/ai/ai-models/services/ai-model-registry.service.ts`.

### Scope limits — what this fix does NOT handle

- **Multi-pod deployments.** Calling `refreshRegistry()` on the pod that handled the GraphQL mutation will not refresh the registry on any other pod in a horizontally-scaled deployment. Those pods will still serve stale registries until a cache-broadcast mechanism (Redis pub/sub / workspace cache-invalidation event) is added. H4 territory.
- **Config variables changed outside the app** (direct SQL on `keyValuePair` / `coreConfigVariables` table, or cron-scheduled external writes). The targeted fix only covers mutations that flow through `admin-panel.resolver.ts`.
- **Changes to the committed `ai-providers.json` catalog (adding/removing a provider key name)** — those require a rebuild anyway, not a fix here.
- **AI_PROVIDERS-shaped custom providers that happen to reference a templated `{{MY_KEY}}` indirection from a custom provider entry** — works today but depends on the same resolveTemplate chain; not a new issue.

### Fix shapes (pragmatism axis)

- **Minimal (parsimonious):** Add three one-line calls. In `admin-panel.resolver.ts`, at the end of `createDatabaseConfigVariable`, `updateDatabaseConfigVariable`, and `deleteDatabaseConfigVariable`, call `this.aiModelRegistryService.refreshRegistry()` (the service is already injected into the resolver at admin-panel.resolver.ts — it's used on lines 406/423/478/511). Gate behind a cheap guard: `if (isAiApiKeyConfigVariable(key)) this.aiModelRegistryService.refreshRegistry();` so unrelated config writes (e.g. `FRONT_BASE_URL`) don't pay the cost. The guard is a small helper that checks whether `key` matches the `apiKey` template of any provider in `loadDefaultAiProviders()`. Est. LOC: ~20 (3 call sites + 1 helper util + 1 unit test). Files: ~3.
- **Idiomatic:** Introduce a typed event (`@OnEvent('config-variable.changed')` or a NestJS `EventEmitter2` signal) emitted by `TwentyConfigService.{set,update,delete}`, with `AiModelRegistryService` subscribing and filtering to the AI-relevant keys. This matches the codebase's existing cross-service decoupling (NestJS already ships `@nestjs/event-emitter` in most Twenty modules) and lets future config-sensitive services (web-search driver, captcha driver, etc.) attach without editing every resolver. Est. LOC: ~60. Files: ~4.
- **Architectural (maximal):** Eliminate the boot-snapshot registry entirely — make `AiModelRegistryService.getAvailableModels()` compute on demand from `ProviderConfigService.getResolvedProviders()` with a short-TTL memoization. Also wire a Redis pub/sub channel for config-variable changes so multi-pod deployments invalidate coherently (closes H4). This is the right long-term shape but significant. Est. LOC: ~300 across the registry, the preferences service (which caches recommended IDs similarly), and the cache-invalidation plumbing. Files: ~8-10.

**Recommended:** **idiomatic** — the minimal fix is a clear regression-patch that closes the reported symptom with trivial risk, but maintainer-review comment #1 below will ask for the generalization. The idiomatic variant is the cheapest way to answer that question up front, removes the asymmetry between the two AI-config write paths (custom-provider mutations vs. config-variable CRUD), and leaves the path clear for a future architectural pass. `codebase.impact` reported LOW risk on `refreshRegistry`, so minimal WOULD be safe-as-patch, but it leaves the bug class alive.

### Maintainer-review self-pass

1. *"This pattern (`resolver writes config → resolver pokes service to refresh`) is duplicated four times for custom providers already and now three more times for config variables. Can we make this event-driven so the AI module owns its own invalidation?"* — exactly the push toward Idiomatic; folded in above.
2. *"What happens if the admin saves a new key in a multi-pod deployment? Does pod B still return zero models?"* — yes, it does (H4). Called out in Scope limits; recommending a Redis-pub/sub follow-up ticket rather than bundling.
3. *"Why didn't the existing test at `ai-models-types.const.spec.ts:132` catch this?"* — because the spec asserts the error **message** for an empty-registry state, not that a post-save `sendChatMessage` succeeds. The fix should add a resolver-level integration test: seed empty registry, call `updateDatabaseConfigVariable("MISTRAL_API_KEY", "sk-test")`, then call `sendChatMessage` and assert success (or at least assert `getAvailableModels().length > 0`).

### Devil's advocate — if the reporter's theory is wrong

Three orthogonal causes to stay honest about: (1) `isProviderConfigured` could return false for saved keys because of a subtle whitespace / quoting issue in the DB value (e.g., the admin panel might be storing the key with surrounding quotes while `resolveTemplate` returns the raw string); this would cause `!!(config.apiKey)` to be true but `createProvider` to throw later — refuted by the code at `is-provider-configured.util.ts:3` and the unchanged `resolveTemplate` path, but a good verification-time sanity-check. (2) `TwentyConfigService.update` could silently no-op if `isDatabaseDriverActive` is false, which would mean the admin panel appears to succeed but the key never persists — ruled out because the code throws `DATABASE_CONFIG_DISABLED` rather than silently accepting, and the user reports no such error at save time. (3) The `SdkProviderFactoryService.createProvider` call at `ai-model-registry.service.ts:89` could fail asynchronously during `buildModelRegistry` for a valid Mistral config, registering zero models even after a refresh — would be visible in server logs as a warning from that service; reporter has not seen such logs. None of these three rises to the confidence of H1.

### Recommended next step

**proceed to fix** — the defect is localized, the fix is small, and the evidence chain is unambiguous. Architect can pick minimal vs. idiomatic in the plan phase; Scout recommends idiomatic.

### Open questions

- Does the deployment actually have `IS_CONFIG_VARIABLES_IN_DB_ENABLED=true`? The workaround ("env vars work") is consistent with either "DB flag is on but the refresh is missing" (H1) or "DB flag is off and save appears to succeed but actually throws" — Scout is betting on the former because the user says the admin panel accepts the save with no error snackbar. A one-line log-level confirmation from the reporter's server would close this.
- Is the reporter running a single pod or multiple? If multiple, the minimal / idiomatic fix still leaves a residual staleness window that may show up as "only some requests fail" after the fix lands. Recommend a follow-up ticket for cross-pod cache invalidation regardless.
- Are there any other config-variable-backed services in the codebase that cache at boot and would benefit from the same invalidation hook (web-search driver, captcha driver, email driver)? The Idiomatic variant answers this structurally; confirm with Architect whether the scope of this PR should include those.

<!-- MACHINE-READABLE FOOTER — DO NOT REMOVE; downstream skills parse this block -->
<!--phoenix:scout-summary
affected_repos: [ShankHarinath/twenty]
root_cause_confidence: 0.92
fix_direction_confidence: 0.90
confidence: 0.92
recommended_next_step: proceed
primary_causal_symbol: packages/twenty-server/src/engine/core-modules/admin-panel/admin-panel.resolver.ts:277
fix_shape_recommended: idiomatic
hypotheses_count: 4
hypotheses_confirmed: 1
converged: true
-->
