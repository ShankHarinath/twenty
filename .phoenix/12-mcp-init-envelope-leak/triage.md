# Triage Report — ShankHarinath/twenty#12

## Summary

The MCP server's `wrapJsonRpcResponse` utility unconditionally merges the `MCP_SERVER_METADATA` constant (`protocolVersion`, `serverInfo`, `metadata`) into every `result` body, and the `McpProtocolService` / `McpToolExecutorService` response builders additionally inject `capabilities` plus empty `resources`/`prompts` arrays into method responses that should not carry them (`tools/list`, `prompts/list`, `resources/list`). The result is that every non-`ping` method returns an `initialize`-shaped envelope. Strict MCP clients (rmcp) reject the extra fields during deserialization. The HTTP `201 Created` is a separate but related symptom: NestJS's default status for `@Post()` without an explicit `@HttpCode(200)` decorator.

Root cause confidence: **0.95**.

## Phase A — Code map

**Repo search radius**: home repo is `ShankHarinath/twenty`. No cross-repo contracts signal (the "client" is the external `rmcp` Rust library, not an indexed Canonix repo). Scope = `twenty` only.

**Files implicated** (all absolute):

- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/api/mcp/controllers/mcp-core.controller.ts`
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/api/mcp/services/mcp-protocol.service.ts`
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/api/mcp/services/mcp-tool-executor.service.ts`
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/api/mcp/utils/wrap-jsonrpc-response.util.ts`
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/api/mcp/constants/mcp.const.ts`
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/src/engine/api/mcp/dtos/json-rpc.ts`
- `/Users/shashank/Canonix/phoenix-test/twenty/packages/twenty-server/test/integration/ai/suites/mcp.controller.integration-spec.ts` (tests currently *assert* the buggy shape — will need to be updated)

**Signal mapping**

| Signal | Location | Notes |
|---|---|---|
| `/mcp` endpoint | `McpCoreController` (`@Controller('mcp')` line 25) | Single `@Post()` handler, no `@HttpCode(200)` → Nest defaults to 201 |
| `initialize` correct | `McpProtocolService.handleInitialize` (lines 51–64) | Intentionally returns capabilities + tools/resources/prompts arrays |
| `tools/list` bug | `McpToolExecutorService.handleToolsListing` (lines 43–74) | Adds `capabilities.tools`, empty `resources: []`, empty `prompts: []` into the result — none of which belong in a spec `tools/list` response |
| `prompts/list` / `resources/list` bug | `McpProtocolService.handleMCPCoreQuery` lines 212–232 | Same pattern — injects `capabilities` into result body |
| Envelope metadata leak | `wrapJsonRpcResponse` (util lines 10–21) | Merges `MCP_SERVER_METADATA` (`metadata`, `protocolVersion`, `serverInfo`) into **every** result/error unless `omitMetadata=true`. Only `ping` sets `omitMetadata=true` (service line 180). |
| HTTP 201 Created | `McpCoreController.handleMcpCore` (lines 31–52) | No `@HttpCode(HttpStatus.OK)`. Confirmed by integration tests expecting `.expect(201)` |
| Missing `Mcp-Session-Id` | Not implemented anywhere | `grep` for `Mcp-Session-Id` across the mcp module returns no results — stateless-only, never emitted |
| Tool names `http_request`, `send_email`, `draft_email`, `search_help_center`, `code_interpreter` | Referenced but excluded from MCP tool set | `MCP_EXCLUDED_TOOLS = new Set(['code_interpreter', 'http_request'])` (protocol service line 39) contradicts user report that `http_request` and `code_interpreter` are returned — but this is orthogonal to the deserialization bug |

**Impact (upstream of `wrapJsonRpcResponse`)**: indexed impact returned 0 direct callers (graph coverage gap for this small util), but manual verification shows 8 call sites across `McpProtocolService` (7) and `McpToolExecutorService` (2). All of them are inside the MCP module — risk LOW, change is localized.

**Affected repos**: `ShankHarinath/twenty` only.

## Phase B — Runtime evidence (kubectl)

Skipped. The reporter's environment is Docker `twentycrm/twenty:v1.18.1`, not the `kind-canonix` cluster; no Twenty deployment is running there. The issue body already contains precise curl-level reproduction showing the exact wire payload, which is stronger evidence than log mining would provide. If the user wants runtime confirmation, they can reproduce against any local `twenty-server` and observe `McpProtocolService.handleMCPCoreQuery` routing through the `tools/list` branch at service lines 208–210.

## Phase C — Recent commits on affected files

```
66da296799  Feb 20 2026  "Unify MCP into single endpoint with lazy tool discovery" (#18113)
e7ebf51e50  Nov 25 2025  "Replace agent handoff system with planning-based router" (#16003)  -- introduced wrap-jsonrpc-response.util.ts and mcp.const.ts
```

- **e7ebf51e50** (#16003) is where the bug was **introduced**: it created `wrap-jsonrpc-response.util.ts` with the unconditional `MCP_SERVER_METADATA` merge, and created `mcp.const.ts` containing `protocolVersion`, `serverInfo`, and `metadata` — the exact extra fields the reporter cites.
- **66da296799** (#18113) consolidated the endpoint and wrote the current `handleToolsListing` / `prompts/list` / `resources/list` branches that duplicate `capabilities` into non-initialize responses. It also added the integration tests that **codify the bug** (see `expect(201)` and `res.body.result.capabilities?.tools?.listChanged`).

The integration-test file is itself part of the fix surface: several assertions need to flip from `.expect(201)` → `.expect(200)` and from "capabilities is defined" → "capabilities is undefined for non-initialize".

## Phase D — Similar past issues / PRs

- `gh issue list --search "MCP tools/list" --state all` on `ShankHarinath/twenty` returns only the present issue (#12). No prior related issues exist in this fork.
- Upstream `twentyhq/twenty` cannot be searched (harness denies owners outside the `ShankHarinath`/`canonix-*` allowlist), so upstream prior art is not known. A manual upstream search is recommended before the fix to avoid duplicate work.

No fix template is available from past incidents; the fix is primarily spec compliance.

## Phase E — Hypothesis and recommended fix shape

**Root cause** — one paragraph:

The response-shaping path is not method-aware. `wrapJsonRpcResponse` blindly merges `MCP_SERVER_METADATA` into every `result`, and the per-method handlers for `tools/list`, `prompts/list`, and `resources/list` each inject a `capabilities` block that the MCP 2024-11-05 spec only expects on `initialize`. The fix must (1) make the metadata merge opt-in rather than opt-out — flip the semantics so only `handleInitialize` carries `MCP_SERVER_METADATA` — and (2) remove the `capabilities`, empty `resources`, empty `prompts` fields from the non-initialize responses so they return spec-compliant shapes (`{ result: { tools: [...] } }`, `{ result: { prompts: [] } }`, `{ result: { resources: [] } }`). Separately, add `@HttpCode(HttpStatus.OK)` to `McpCoreController.handleMcpCore` to return 200 for success, and emit an `Mcp-Session-Id` response header (can be a stable UUID per request, even in stateless mode) to satisfy the Streamable HTTP transport expectations. The integration test suite at `mcp.controller.integration-spec.ts` must be updated in lockstep — its current assertions lock in the buggy behaviour.

**Suggested code shape** (for Architect — not for Scout to apply):

1. `wrap-jsonrpc-response.util.ts` — invert `omitMetadata` to `includeServerMetadata = false` (default off); only `handleInitialize` passes `true`.
2. `mcp-protocol.service.ts` —
   - `tools/list` branch does not need `capabilities`/`resources`/`prompts` injected (currently it calls `handleToolsListing` which does this).
   - `prompts/list` returns `{ result: { prompts: [] } }` only.
   - `resources/list` returns `{ result: { resources: [] } }` only.
3. `mcp-tool-executor.service.ts::handleToolsListing` — result body becomes `{ tools: toolsArray }` only.
4. `mcp-core.controller.ts` — add `@HttpCode(HttpStatus.OK)` on the `@Post()` handler. Optionally add `@Header('Mcp-Session-Id', '<uuid>')` or set it inside the handler via `@Res()`.
5. `mcp.controller.integration-spec.ts` — flip `.expect(201)` → `.expect(200)`; remove assertions of `capabilities` / `resources` / `prompts` on non-initialize responses; keep them for `initialize` only.

**Confidence**: **0.95**. Evidence is end-to-end:
- user's curl output matches exactly what the code constructs;
- the util's merge semantics are unambiguous (5 lines);
- the integration tests document the buggy shape, proving the symptom isn't an environment artefact;
- git-blame points cleanly to #16003 (introduction) and #18113 (extension);
- blast radius is entirely inside `packages/twenty-server/src/engine/api/mcp/`.

## Open Questions

1. The user reports `http_request` and `code_interpreter` *are* returned by `tools/list`, but `MCP_EXCLUDED_TOOLS` in `mcp-protocol.service.ts` excludes both. Either the exclusion is bypassed somewhere (possibly via `COMMON_PRELOAD_TOOLS`) or the reporter was on a slightly different version. This is a **secondary** question and does not block the primary fix, but Architect should verify against v1.18.1 source whether the exclusion was added later than the reported version.
2. The MCP spec (2024-11-05 Streamable HTTP) allows omitting `Mcp-Session-Id` in pure-stateless servers — confirm whether the rmcp client treats it as a hard requirement or just a strong preference. If preference only, the 201→200 and shape fixes may be sufficient to unblock the reporter without session-id work.
3. Upstream `twentyhq/twenty` may already have a fix — harness blocks cross-owner `gh` search. Architect should manually check upstream before authoring the patch.

## Recommended next step

Dispatch `/phoenix:plan` against `ShankHarinath/twenty` to author the fix. Scope is a single repo, ~5 files, with a well-defined spec target (MCP 2024-11-05). LOW risk, estimated small diff.

---

```yaml
# machine-readable
affected_repos:
  - ShankHarinath/twenty
confidence: 0.95
recommended_next_step: plan
```
