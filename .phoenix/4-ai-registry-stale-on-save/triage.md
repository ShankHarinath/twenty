# Triage Report — ShankHarinath/twenty#4

## Summary

On self-hosted Twenty, API keys saved via Settings → AI → Models → *Configure API Key* are persisted as **database-backed config variables** (e.g. `MISTRAL_API_KEY`) but the server's in-memory `AiModelRegistryService` is never rebuilt after that save, so `getAvailableModels()` stays empty until the next server restart. The user-visible error `"No AI models are available."` comes from `AiModelRegistryService.getDefaultModelForRole()`.

Environment-variable keys work because `AiModelRegistryService` is rebuilt once in its constructor (application bootstrap), at which point `process.env.MISTRAL_API_KEY` is already present — exactly as the reporter observed.

## Phase A — Code map

Repos discovered via `list_repos`: `twenty` (17102 files, 75867 nodes, embeddings=0 — text/graph fallback only), plus several unrelated repos. Only `twenty` is relevant; no group membership.

Signal → code:

| Signal | Resolved symbol | File |
|---|---|---|
| `"No AI models are available."` (literal) | `AiModelRegistryService.getDefaultModelForRole` L186-205 (throws the `AgentException` at L198-201) | `packages/twenty-server/src/engine/metadata-modules/ai/ai-models/services/ai-model-registry.service.ts` |
| "model registry" | `AiModelRegistryService` | same file, L44-378 |
| "AI Chat" entry point (server) | `AiGenerateTextController` (`POST /generate-text`) | `packages/twenty-server/src/engine/metadata-modules/ai/ai-generate-text/controllers/ai-generate-text.controller.ts` |
| Admin panel AI provider UI | `SettingsAdminAiProviderDetail` | `packages/twenty-front/src/pages/settings/admin-panel/SettingsAdminAiProviderDetail.tsx` |
| Config-variable save hook (UI) | `useConfigVariableActions` | `packages/twenty-front/src/modules/settings/admin-panel/config-variables/hooks/useConfigVariableActions.ts` |
| DB config-variable mutations (server) | `AdminPanelResolver.createDatabaseConfigVariable` / `updateDatabaseConfigVariable` / `deleteDatabaseConfigVariable` L264-295 | `packages/twenty-server/src/engine/core-modules/admin-panel/admin-panel.resolver.ts` |
| Template resolver (`{{MISTRAL_API_KEY}}` → actual key) | `ProviderConfigService.resolveTemplate` / `getResolvedProviders` | `packages/twenty-server/src/engine/metadata-modules/ai/ai-models/services/provider-config.service.ts` |
| Registry refresh entry point (only 4 callers) | `AiModelRegistryService.refreshRegistry` L358-360 | `ai-model-registry.service.ts` |

Graph search on `refreshRegistry` callers (grep + context): invoked only from `admin-panel.resolver.ts` L406 (`addAiProvider`), L423 (`removeAiProvider`), L478 (`addModelToProvider`), L511 (`removeModelFromProvider`). **Not** called from `createDatabaseConfigVariable`, `updateDatabaseConfigVariable`, `deleteDatabaseConfigVariable`, `setModelAdminEnabled`, `setModelRecommended`, or `setDefaultModel`.

## Phase B — Runtime evidence

Not applicable. The reporter is on Railway (external self-hosted), not the local `kind-canonix` cluster. The error message in the issue (`"No AI models are available"`) matches the literal string thrown by `getDefaultModelForRole` at `ai-model-registry.service.ts:199` — a code-level identity match, stronger than a log grep.

## Phase C — Recent commits of interest

- `908aefe7c1` (#18818, 2026-03-21, Félix Malfait) — **"feat: replace hardcoded AI model constants with JSON seed catalog"**. Introduces `ProviderConfigService`, the catalog-based provider model, and the `AiModelRegistryService` with its single-shot `buildModelRegistry()` in the constructor. This is the commit that established the current architecture; the bug has existed since.
- `7a341c6475` (#19016) — adds `authType` support; does not change the refresh lifecycle.
- No later commit adds `refreshRegistry()` wiring to the config-variable mutations.

`git blame` on `ai-model-registry.service.ts:197-201` traces back to 908aefe7c1 as well. The reporter's issue is a direct consequence of that refactor: the four provider-shape mutations were wired to refresh, but the *value* mutations (`*DatabaseConfigVariable`) were not.

## Phase D — Similar past issues / PRs

Searched `gh issue list` and `gh pr list` on `ShankHarinath/twenty` for `"No AI models are available"`, `refreshRegistry`, `AI provider admin panel`: zero hits. This appears to be the first report.

## Phase E — Root cause hypothesis

**Root cause.** For catalog providers (Mistral, OpenAI, Anthropic, etc.) the admin-panel UI in `SettingsAdminAiProviderDetail.tsx` L209-229 does not edit the provider entry; it links the admin to the generic config-variable page, where the Mistral API key is saved via `createDatabaseConfigVariable('MISTRAL_API_KEY', '…')` → `AdminPanelResolver.createDatabaseConfigVariable` → `TwentyConfigService.set()` → `DatabaseConfigDriver.set()` (updates DB *and* cache).

But `AdminPanelResolver.createDatabaseConfigVariable` / `updateDatabaseConfigVariable` / `deleteDatabaseConfigVariable` (L264-295) return without calling `this.aiModelRegistryService.refreshRegistry()`. The registry therefore still holds the state from application boot: for Mistral, `isProviderConfigured(config)` was `false` (apiKey template `{{MISTRAL_API_KEY}}` resolved to `undefined`), so no `sdkInstance` was created (L88-90 of the registry) and no entries were added to `modelRegistry` for that provider (L105-114). Subsequent `getAvailableModels()` is still empty → `getDefaultModelForRole` throws `"No AI models are available."` (L197-201).

Env-var workaround works because at application startup `ProviderConfigService.resolveTemplate` reads `process.env[varName]` (L73) and the registry constructor builds a populated `modelRegistry`.

The full provider-replace mutations (`addAiProvider`, `removeAiProvider`, `addModelToProvider`, `removeModelFromProvider`) already call `refreshRegistry()`, which is why *custom* providers work — they set the entire `AI_PROVIDERS` structure including a literal apiKey value, and explicitly refresh.

**Confidence: 0.9.** Literal error-string match, single-file registry, graph-verified caller set for `refreshRegistry` (grep: exactly 4 callers, none of them config-variable mutations), resolver code read end-to-end, UI flow traced from `SettingsAdminAiProviderDetail` → config-variable page → `useConfigVariableActions` → `createDatabaseConfigVariable` mutation. The only residual uncertainty is whether there's an out-of-band refresh mechanism (e.g., a DB-change listener) I haven't seen — a broad grep did not surface one.

## Recommended fix (for Architect)

Call `refreshRegistry()` from the three DB-config-variable mutations in `admin-panel.resolver.ts` whenever the affected key is one that participates in AI provider template resolution. Two viable shapes:

1. **Minimal / narrow** — in `createDatabaseConfigVariable`, `updateDatabaseConfigVariable`, `deleteDatabaseConfigVariable`, after the await, if the key matches any `apiKeyConfigVariable` referenced by the resolved catalog (available via `providerConfigService.getResolvedProviders()` or a new helper), call `this.aiModelRegistryService.refreshRegistry()`. Avoids refreshing on unrelated variable writes.

2. **Unconditional** — always call `refreshRegistry()` at the end of those three mutations. `refreshRegistry()` just clears three Maps and re-walks the catalog (O(providers × models), handful of items); cheap enough that unconditional refresh on any admin-panel DB-config write is defensible. Simpler, fewer edge cases.

Either way, `resolveModelForAgent` / `validateModelAvailability` stay unchanged — they already read through the registry. No schema or migration changes.

Blast radius on `refreshRegistry` is LOW (`impact` d=0, and only internal calls to `buildModelRegistry` which clears and rebuilds maps).

Suggested additional test:
- Unit test on `AdminPanelResolver`: after `updateDatabaseConfigVariable('MISTRAL_API_KEY', '…')`, `aiModelRegistryService.getAvailableModels()` should include Mistral models (once `AI_PROVIDERS` catalog has an entry with `apiKeyConfigVariable: MISTRAL_API_KEY`).

## Files Architect will likely touch

- `packages/twenty-server/src/engine/core-modules/admin-panel/admin-panel.resolver.ts` (L264-295) — add `refreshRegistry()` call (and ensure `AiModelRegistryService` is injected; per L78 it already is).
- Optional helper: `packages/twenty-server/src/engine/metadata-modules/ai/ai-models/services/provider-config.service.ts` — add a method to enumerate "api-key-bearing config variables" if going with the narrow variant.
- Test file: `packages/twenty-server/src/engine/core-modules/admin-panel/__tests__/admin-panel.resolver.spec.ts` (or co-located) — new case.

## Open questions

- Is there an intended out-of-band refresh (e.g., a `ConfigVariableChanged` event handler)? Grep found none; worth Architect confirming there isn't a planned event bus approach they'd rather reuse.
- Should setting `MISTRAL_API_KEY = ''` (delete path) also immediately drop Mistral models from the registry? Yes — applying the fix symmetrically to `deleteDatabaseConfigVariable` achieves this; worth calling out in the PR.
- GitNexus index for `twenty` has `embeddings: 0`, so only text / graph search was used; I've cross-verified each finding by reading the files directly.
- `kubectl` Phase B was skipped (external host); no runtime-log cross-check possible.

## Machine-readable footer

<!--phoenix:scout-summary
affected_repos:
  - ShankHarinath/twenty
confidence: 0.9
recommended_next_step: plan
-->
