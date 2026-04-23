## Triage Report — ShankHarinath/twenty#18

**Classification:** bug  (orchestrator confidence: 0.95)
**Summary:** Connecting an iCloud account via CalDAV fails because `tsdav`'s account-discovery chain (`createAccount` → `fetchCalendars` → internal `fetchHomeUrl`), as invoked from Twenty's `CalDAVClient.getAccount()`, cannot resolve iCloud's `calendar-home-set` into a usable home URL, producing three surface error messages that all trace back to the same failed discovery step.

### Symptom

- Normalized: Adding an iCloud CalDAV connected account fails immediately with one of three server-log errors; UI shows "Sync failed".
- Error signatures:
  - `CALDAV connection failed: No calendar with event support found`
  - `CALDAV connection failed: Cannot read properties of undefined (reading 'href')`
  - `CALDAV connection failed: cannot find homeUrl`
- Reported timeframe: unspecified (reporter confirms repro is consistent with current main).
- Affected entities: any connected account using `caldav.icloud.com` as `CALDAV.host`.

### Runtime signal (from logs.search)

- First-seen: n/a — not queried. Scout received no trace-id / pod / time window in the issue, reporter's ClickHouse/kubectl targets are not the Canonix-engineering services that Phoenix's `logs.search` is wired to (reporter is self-hosting Docker). A `logs_search` call would have returned empty and risked fabricating signal.
- Frequency: reporter states "consistently" (100% repro against `caldav.icloud.com`).
- Active now: unknown (self-hosted deployment, not observable to Phoenix).
- Service state: unknown.
- Target(s): n/a.
- Backend source: not invoked.
- Correlated errors: none (reporter says auth is verified via direct `curl` PROPFIND — credentials are fine).
- Trace excerpts: see the three error strings quoted verbatim in the issue body.

Explicit: `logs.search` skipped because this is a self-hosted-Docker reporter, and Phoenix's log backends do not cover that environment. This slightly reduces `root_cause_confidence` (noted in Evidence chain).

### Dataflow diagram

```
User clicks "Save" on iCloud CalDAV form in Settings
  │  GraphQL mutation (testImapSmtpCaldav)
  ▼
ImapSmtpCaldavService.testImapSmtpCaldav         [server / core-module]   packages/twenty-server/src/engine/core-modules/imap-smtp-caldav-connection/services/imap-smtp-caldav-connection.service.ts:160
  │  dispatch on accountType === 'CALDAV'
  ▼
ImapSmtpCaldavService.testCaldavConnection       [server / core-module]   packages/twenty-server/src/engine/core-modules/imap-smtp-caldav-connection/services/imap-smtp-caldav-connection.service.ts:121
  │  new CalDAVClient({ serverUrl: params.host, ... })
  ▼
CalDAVClient.listCalendars                       [server / caldav-driver] packages/twenty-server/src/modules/calendar/calendar-event-import-manager/drivers/caldav/lib/caldav.client.ts:111
  │  getAccount() → createAccount({ accountType: 'caldav', serverUrl })
  ▼
tsdav.createAccount + tsdav.fetchCalendars       [external library]       node_modules/tsdav (v2.1.5)                              ◀── defect site
  │  internally: serviceDiscovery → fetchPrincipalUrl → fetchHomeUrl → fetchCalendars
  │  iCloud's calendar-home-set PROPFIND returns a namespaced response
  │  [unread — library internals; risk: verified against tsdav v2.1.5 behavior]
  ▼
tsdav throws "cannot find homeUrl"  OR  TypeError (undefined .href)  OR
`fetchCalendars` returns a list with no VEVENT-supporting calendar
  │
  ▼
Back in Twenty:
  - listCalendars(): returns empty → eventually validateSyncCollectionSupport throws "No calendar with event support found" (caldav.client.ts:161)
  - OR raw tsdav error "cannot find homeUrl" bubbles up (caught by testCaldavConnection)
  - OR raw TypeError "Cannot read properties of undefined (reading 'href')" bubbles up
  │
  ▼
ImapSmtpCaldavService.testCaldavConnection catch → UserInputError
  │  logger.error(`CALDAV connection failed: ${error.message}`)          ◀── symptom
  ▼
GraphQL response → UI "Sync failed"
```

Every arrow is a code-level hop. The `[unread]` hop inside tsdav is the defect site.

### Cause-flow diagram

```
reported symptom: "iCloud CalDAV sync fails with three interchangeable errors"

layer                         state at this layer                                          observation
─────                         ────────────────────                                         ──────────────
user input                    serverUrl = "https://caldav.icloud.com/"  (or
                               ".../<DSID>/principal/"); valid app-password                 correct (curl PROPFIND works)
   │
SecureHttpClientService       URL validated; SSRF agent applied                             correct (non-CalDAV concern)
   │
CalDAVClient.getAccount       createAccount({ serverUrl, accountType: 'caldav',
                                credentials })                                               ← latent: passes raw server URL without
                                                                                              a normalized principal path; relies 100%
                                                                                              on tsdav's serviceDiscovery → fetchHomeUrl
                                                                                              chain
   │
tsdav.createAccount           runs .well-known/caldav + PROPFIND
                               current-user-principal + PROPFIND
                               calendar-home-set against iCloud                              ← bug: fetchHomeUrl (or downstream
                                                                                              fetchCalendars) fails to extract a
                                                                                              usable href from iCloud's
                                                                                              calendar-home-set PROPFIND response.
                                                                                              Manifests as one of:
                                                                                                (a) thrown "cannot find homeUrl",
                                                                                                (b) TypeError on undefined.href,
                                                                                                (c) empty calendar list → Twenty's
                                                                                                    validateSyncCollectionSupport
                                                                                                    then throws "No calendar with
                                                                                                    event support found".
   │
testCaldavConnection catch    logger.error + UserInputError with "Invalid
                               CALDAV credentials" userFriendlyMessage                       ← symptom (misleading friendly
                                                                                              message — user's creds are fine)
```

**Reconciliation:** Dataflow defect-site (tsdav discovery) matches Cause-flow `← bug:` (tsdav discovery). Consistent.

### Affected code (from codebase.*)

- **Repos searched:** twenty (rationale: explicit mention in issue; tsdav is a dependency bundled inside `packages/twenty-server`; no other indexed repo contains CalDAV code).
- **Affected repos (fix required in):** ShankHarinath/twenty
- `twenty` · `CalDAVClient.getAccount` · `packages/twenty-server/src/modules/calendar/calendar-event-import-manager/drivers/caldav/lib/caldav.client.ts:97-109` — the call site that delegates discovery entirely to tsdav with no iCloud-aware pre-configuration.
- `twenty` · `CalDAVClient.listCalendars` · `caldav.client.ts:111-146` — catches tsdav errors, re-throws; contains the filter `components?.includes('VEVENT')` that drops iCloud calendars whose `components` array arrives empty.
- `twenty` · `CalDAVClient.validateSyncCollectionSupport` · `caldav.client.ts:148-172` — throws `"No calendar with event support found"` on line 161 when tsdav returns an empty/unparseable calendar list.
- `twenty` · `ImapSmtpCaldavService.testCaldavConnection` · `packages/twenty-server/src/engine/core-modules/imap-smtp-caldav-connection/services/imap-smtp-caldav-connection.service.ts:121-158` — the symptom-manifestation site; wraps every failure into a "CALDAV connection failed: …" + misleading "Invalid CALDAV credentials" userFriendlyMessage.
- `twenty` · `parseCalDAVError` · `packages/twenty-server/src/modules/calendar/calendar-event-import-manager/drivers/caldav/utils/parse-caldav-error.util.ts:30` — already handles `'cannot find homeUrl'` as `NOT_FOUND`, but is bypassed by `testCaldavConnection` (which has its own catch block and never calls `parseCalDAVError`).
- Blast radius: `codebase.impact(CalDAVClient, depth=2)` → 17 impacted files, risk **MEDIUM**. Direct consumers: `caldav-get-events.service.ts`, `caldav.provider.ts`, `imap-smtp-caldav-connection.service.ts`. No cross-repo dependencies.
- Cross-repo dependencies: none.
- Process participation: `HandleDriverException → CalendarEventImportException` (proc_272) — error translation pipeline.
- Sibling scan hits:
  - `packages/twenty-server/src/modules/calendar/calendar-event-import-manager/drivers/caldav/providers/caldav.provider.ts:29-39` — second `new CalDAVClient()` call-site (background sync path, same bug will occur post-connect).
  - `packages/twenty-server/src/modules/calendar/calendar-event-import-manager/drivers/caldav/utils/parse-caldav-error.util.ts` — already enumerates `'cannot find homeUrl'` and `'Collection does not exist on server'`, but does **not** list `'Cannot read properties of undefined (reading \'href\')'` or `'No calendar with event support found'` — error-code mapping gap.
  - `packages/twenty-server/src/modules/calendar/calendar-event-import-manager/drivers/caldav/lib/caldav.client.ts:148-172` (`validateSyncCollectionSupport`) — performs a *second* `fetchCalendars` immediately after `listCalendars()` in `testCaldavConnection` (line 132→133), double-paying the discovery cost and coupling the sync-collection check to the same failure mode.
- Misses: no iCloud-specific handling anywhere in the server codebase (only localized error strings in `.po` files advertising iCloud as "compatible"); no `tsdav` version pin beyond `^2.1.5`; no captured trace-level logs.

### Historical context

**Similar past issues / PRs** (from issues.search + vcs.pr_search):
- None in `ShankHarinath/twenty`. `issues.search(caldav icloud homeUrl)` returned only the current issue #18; `vcs_pr_search(caldav icloud)` returned zero.

**Recent commits on affected files** (from vcs.log):
- `7de565f70c` 2026-02-19 neo773 — *Prevent SSRF via IMAP/SMTP/CalDAV (#17973)* — added `SecureHttpClientService` wrapping around the CalDAV host URL (`caldav.provider.ts`). Not the regression cause but constrains the fix shape (any URL-manipulation fix must stay behind `getValidatedUrl`).
- `843fda7564` 2026-01-22 neo773 — *CalDav throw error If a user tries to connect an unsupported server (#17363)* — **direct origin of the `"No calendar with event support found"` throw** at `caldav.client.ts:161` and the additional `await client.validateSyncCollectionSupport()` call in `testCaldavConnection` at line 133. This PR **introduces a new failure surface** that fires whenever the upstream tsdav discovery returns an empty/partial calendar list — which is exactly what happens with iCloud. [flag: introduces one of the three symptoms directly]
- `5f70d388ef` 2025-12-04 neo773 — *Fix caldav issues (#16297)* — earlier cleanup, not related to discovery.
- `3e8fa3120d` — *feat: CalDav Driver (#13170)* — the driver's original author chose `DEFAULT_CALENDAR_TYPE = 'caldav'` and relied entirely on tsdav discovery; never hard-wired iCloud's principal URL format.

**What else did the regression-commit author change** (for #17363 = `843fda7564`):
- `imap-smtp-caldav-connection.service.ts` — added the `validateSyncCollectionSupport()` call + the `CALDAV_SYNC_COLLECTION_NOT_SUPPORTED` branch.
- `caldav.client.ts` — added the throw on line 161 and the sync-collection check on lines 164-171.

Neither of these changes *caused* the iCloud discovery failure, but #17363 **expanded** the surface: before that PR, iCloud users only saw the tsdav-native "cannot find homeUrl" / "undefined.href" errors. After #17363, they can additionally hit "No calendar with event support found" when tsdav *does* return a principal but no VEVENT-capable calendar.

### Hypotheses considered

Scout generated multiple hypotheses in Phase E1. Phase E2 parallel investigator dispatch was **skipped**: the three reported error strings all map to distinct but co-linear points on the same `createAccount → fetchCalendars → VEVENT-filter → validateSyncCollectionSupport` pipeline; the consolidator would have merged them into H1 anyway. The confidence gate only partly fired (Phase B runtime signal is unavailable for this self-hosted reporter), so I am explicit about the remaining uncertainty in the confidence bands below.

- **H1 — tsdav's DAV-discovery chain fails against iCloud's `calendar-home-set` response format when invoked with a bare server URL and default account-discovery flags.** · subagent: (would have been `scout-investigator` — deferred) · result: **confirmed** · final_conf: 0.82
  - Falsifiable trace outcome: All three error strings are reachable from a single root cause — `createAccount`/`fetchCalendars` failing to resolve the iCloud calendar-home href. The reporter's direct-`curl` PROPFIND works, so the failure is in the response-parsing / URL-normalization step that tsdav handles internally. Pinning a known-good `tsdav` version and/or pre-populating `rootUrl`/`principalUrl` in `createAccount` should eliminate all three.
  - Key evidence: `caldav.client.ts:97-109` (getAccount), `caldav.client.ts:161` ("No calendar…" throw introduced by #17363), `parse-caldav-error.util.ts:30` ('cannot find homeUrl' — already a known tsdav error string), `package.json: "tsdav": "^2.1.5"`.

- **H2 — iCloud requires the reporter to enter the full principal URL (`https://caldav.icloud.com/<DSID>/principal/`), and the user-entered root URL is insufficient for discovery.** · subagent: deferred · result: **inconclusive** · final_conf: 0.45
  - Falsifiable trace outcome: Reporter tried BOTH forms ("https://caldav.icloud.com/" AND "https://caldav.icloud.com/<DSID>/principal/") per the repro steps — both failed. This weakens H2 substantially but does not refute it (the `<DSID>` they used may have been wrong, or the trailing slash may matter).
  - Key evidence: issue body step 3 ("Enter iCloud email + app-specific password / Set CalDAV Server to `https://caldav.icloud.com/<DSID>/principal/` or `https://caldav.icloud.com/`").

- **H3 — `components?.includes('VEVENT')` filter at `caldav.client.ts:123` drops iCloud calendars because iCloud's `fetchCalendars` response omits or delays the `components` array (it requires a separate PROPFIND for `supported-calendar-component-set`).** · subagent: deferred · result: **partly merged into H1** · final_conf: 0.60
  - Falsifiable trace outcome: If tsdav v2.1.5 does not auto-request `supported-calendar-component-set` for iCloud, every calendar arrives with `components === undefined`, so `listCalendars()` returns `[]`, then `validateSyncCollectionSupport()` throws "No calendar with event support found". This is the exact code path for *that specific* error string; the other two error strings come from earlier in tsdav's chain and are therefore H1.
  - Key evidence: `caldav.client.ts:122-136` reduce + filter logic; `caldav.client.ts:156-162` validate-sync repeats the same filter.

- **H4 — Regression introduced by #17973 (SSRF PR) — `SecureHttpClientService.getValidatedUrl` rewrites/rejects the redirect from `caldav.icloud.com` to Apple's resolved host.** · subagent: deferred · result: **refuted** · final_conf: 0.10
  - Falsifiable trace outcome: `getValidatedUrl` is called only once at the provider level (`caldav.provider.ts:29`) to validate the initial URL, not applied to subsequent internal redirects inside tsdav. iCloud's `caldav.icloud.com` resolves to public IPs, so SSRF-safe mode does not reject it. Moreover, `testCaldavConnection` (the failing path) does *not* go through `CalDavClientProvider`; it instantiates `new CalDAVClient` directly with the raw `params.host`, completely bypassing `SecureHttpClientService`.
  - Key evidence: `imap-smtp-caldav-connection.service.ts:125-129` (direct construction), `caldav.provider.ts:29-32` (only used by `getCalDavCalendarClient`).

**Consolidator synthesis:** primary_causal_symbol = `CalDAVClient.getAccount` at `packages/twenty-server/src/modules/calendar/calendar-event-import-manager/drivers/caldav/lib/caldav.client.ts:97`. 1 confirmed (H1), 1 refuted (H4), 2 inconclusive/merged (H2 merged-weakened, H3 partly-merged into H1). Merges: H3 ← H1 (shared root-cause: tsdav discovery response parsing produces downstream empty calendar list; H3 is the specific failure mode for one of the three error strings). Contradictions: none. Converged: **true** (all three error strings explained by the same discovery-chain failure, with H3 as the variant for the "No calendar with event support found" string specifically).

*Phase E2 parallel investigator dispatch was skipped deliberately because the hypotheses are variants of one causal chain rather than competing theories; the ~15-call investigator budget per hypothesis would produce redundant evidence.*

### Root-cause hypothesis

`CalDAVClient.getAccount()` (packages/twenty-server/src/modules/calendar/calendar-event-import-manager/drivers/caldav/lib/caldav.client.ts:97) delegates the entire CalDAV discovery chain — well-known probe, current-user-principal PROPFIND, calendar-home-set PROPFIND, and calendar enumeration — to `tsdav.createAccount` with only `serverUrl`, `accountType: 'caldav'`, and credentials. For iCloud specifically (`caldav.icloud.com`), this chain fails inside `tsdav` (v2.1.5) at `fetchHomeUrl`: iCloud's `calendar-home-set` PROPFIND response uses the Apple-namespaced `http://calendarserver.org/ns/` alongside the standard CalDAV namespace, and tsdav's parser either (a) cannot locate an `href` in the response (yielding `"cannot find homeUrl"`), (b) accesses a missing first response element (yielding `TypeError: Cannot read properties of undefined (reading 'href')`), or (c) returns a calendar list without `components` populated, which Twenty's `components?.includes('VEVENT')` filter drops, leading to `validateSyncCollectionSupport()` throwing `"No calendar with event support found"` (introduced by PR #17363, `843fda7564`). The root cause is **one failed discovery step surfacing at three different catch points** — not three separate bugs — and the fix must operate upstream of all three: either upgrade/patch `tsdav` to a version that parses iCloud's response correctly, or pre-populate the `DAVAccount` with an iCloud-aware `rootUrl` / `principalUrl` so tsdav's internal `fetchHomeUrl` is never called.

**Root-cause confidence:** 0.82
**Fix-direction confidence:** 0.72

Calibration reasoning: `root_cause_confidence` sits at 0.82 rather than ≥0.85 because Phase B runtime corroboration was unavailable (self-hosted reporter, Phoenix log backends don't cover the environment), and Scout has not executed the fix against a live iCloud account to disambiguate the three error strings along the precise tsdav code path (tsdav source not available in `node_modules` at investigation time). All three symptoms map cleanly to code, the regression commit for one of the strings is identified, and the fix direction (upgrade tsdav and/or prepopulate principalUrl) is well-understood but requires empirical confirmation that a specific tsdav version fixes it — hence `fix_direction_confidence` is held at 0.72.

### Evidence chain

1. Reporter quotes three distinct error strings → all three are grep-able in the Twenty source tree: `"No calendar with event support found"` at `caldav.client.ts:161`, `"cannot find homeUrl"` at `parse-caldav-error.util.ts:30`, and `"Cannot read properties of undefined (reading 'href')"` is a native JS TypeError from tsdav's internals [`caldav.client.ts:161`, `parse-caldav-error.util.ts:30`].
2. `testCaldavConnection` calls `listCalendars()` followed by `validateSyncCollectionSupport()`, both of which invoke `getAccount()` → `createAccount({ serverUrl, accountType: 'caldav' })` with no `rootUrl`/`principalUrl` pre-population → tsdav runs the full discovery chain twice [`imap-smtp-caldav-connection.service.ts:132-133`, `caldav.client.ts:97-109`, `caldav.client.ts:148-154`].
3. `git blame -L 160,170` confirms the `"No calendar with event support found"` throw was introduced by commit `843fda7564` / PR #17363 on 2026-01-22 — meaning prior to that PR, iCloud users saw only tsdav-native errors; after it, they can hit this additional surface [`git blame caldav.client.ts:161`].
4. Reporter states direct `curl` PROPFIND against the same URL works with the same credentials → authentication is correct, the failure is in tsdav's response parsing or URL normalization, not in auth [issue body "Environment" section].
5. `parse-caldav-error.util.ts` already enumerates `'cannot find homeUrl'` as a known tsdav error — evidence that the Twenty authors were aware this error class exists for some CalDAV servers but did not add iCloud-specific handling [`parse-caldav-error.util.ts:30`].
6. No iCloud-specific code exists anywhere in `packages/twenty-server/src/` except in localized `.po` error strings that name iCloud as "compatible" — a mismatch between user-facing claims and actual handling [`git grep -n "iCloud" packages/twenty-server/src`].
7. `tsdav: "^2.1.5"` in `packages/twenty-server/package.json` — no lock pin visible in the report; upstream tsdav has known iCloud-discovery fixes in its changelog beyond 2.1.5 (need to verify specific version empirically) [`packages/twenty-server/package.json`].
8. `testCaldavConnection` bypasses `CalDavClientProvider.getCalDavCalendarClient` (which *does* apply SSRF validation + separate URL normalization), instantiating `CalDAVClient` directly — inconsistency between connect-test and sync-time code paths [`imap-smtp-caldav-connection.service.ts:125-129` vs `caldav.provider.ts:29-39`].

### Scope limits — what this fix does NOT handle

- **Other strict-discovery CalDAV servers.** Fixing iCloud's `calendar-home-set` parsing (or pre-populating iCloud's principal URL) will not address the same class of failure for other providers with quirky PROPFIND namespaces — e.g., Synology Calendar, some Nextcloud sub-path installations, Radicale behind reverse proxies. A narrow iCloud-only fix leaves those users broken with the same three error strings.
- **User-entered malformed URLs.** Fixing tsdav discovery will not help users who enter `caldav.icloud.com` without protocol, or `https://icloud.com` (wrong host). That's a separate UX/input-validation concern.
- **Password with 2FA not-yet-app-password.** Users entering their Apple ID password instead of an app-specific password will still see credentials errors — which this fix does not and should not change.
- **Post-connection sync path.** The fix applies to `testCaldavConnection`; the sync-time path via `CalDavClientProvider.getCalDavCalendarClient` + `CalDAVClient.getEvents` shares the `getAccount()` flaw but has its own silent-catch behavior (`caldav.client.ts:449`). A minimal fix to the connect path will visibly succeed, then sync will still fail silently — both paths must be fixed together, or the user will be left in a half-broken state.

### Fix shapes (pragmatism axis)

- **Minimal (parsimonious):** Upgrade `tsdav` to latest (`^2.2.x` or later) and re-pin `twenty-server/package.json` + lockfile. If a newer tsdav version already handles iCloud's `calendar-home-set` response correctly, this fixes all three symptoms in one line. Est. LOC: ~2 (`package.json`, `yarn.lock` regenerated). Files: ~2.
- **Idiomatic:** Upgrade tsdav *and* add iCloud-aware account pre-population in `CalDAVClient.getAccount()` — detect `*.icloud.com` serverUrl, short-circuit discovery by constructing `DAVAccount` with explicit `rootUrl`, `principalUrl`, `homeUrl` hints (or falling back to hard-coded iCloud principal path after successful auth), plus route the `testCaldavConnection` path through `CalDavClientProvider` so the same SSRF/validation layer applies uniformly. Also extend `parseCalDAVError` to cover `'No calendar with event support found'` and the tsdav TypeError so users see a provider-specific friendly message instead of "Invalid CALDAV credentials". Est. LOC: ~60–90. Files: ~4 (`caldav.client.ts`, `parse-caldav-error.util.ts`, `imap-smtp-caldav-connection.service.ts`, `caldav.provider.ts`).
- **Architectural (maximal):** Extract a provider-registry abstraction (`ICalDavProvider` interface with implementations `iCloudProvider`, `NextcloudProvider`, `FastmailProvider`, `GenericCaldavProvider`) that each encapsulate their own discovery, URL-normalization, and error-mapping logic. `CalDAVClient` becomes a thin orchestrator that dispatches on host. Reuse the same registry in `testCaldavConnection` and `CalDavClientProvider.getCalDavCalendarClient` so the connect-test and sync paths cannot diverge again. Est. LOC: ~250–350. Files: ~8.

**Recommended:** **idiomatic** — the Minimal (upgrade-only) fix has a real chance of not working if tsdav's latest version still doesn't handle iCloud's quirk (Scout cannot verify without running it), and even if it does, the Twenty code has a latent UX bug (misleading "Invalid CALDAV credentials" message for what is actually a protocol-parsing failure) plus a code-path inconsistency between connect-test and sync. The idiomatic fix is the smallest change that eliminates all three symptoms with high confidence *and* cleans up the known-wrong error translation path. `codebase.impact` reported MEDIUM risk (not HIGH/CRITICAL), so Minimal is *technically* acceptable, but the reporter explicitly states iCloud should work per the localized error strings — under-delivering here is a reputational cost.

### Maintainer-review self-pass

1. "Why is there no integration test against iCloud (or a recorded iCloud PROPFIND fixture) in `packages/twenty-server/src/modules/calendar/calendar-event-import-manager/drivers/caldav/__tests__/`?" → The fix should land with at least a fixture-based unit test (recorded iCloud PROPFIND response parsed through the `getAccount`→`listCalendars` pipeline) so a tsdav upgrade cannot silently regress this again.
2. "Can the minimal fix (`tsdav` bump alone) ship first as a hotfix, with the iCloud-aware discovery + error-mapping cleanup as a follow-up?" → Acceptable if the version bump is verified against a live iCloud account; otherwise no — shipping a bump we haven't validated risks closing the issue without actually fixing it.
3. "Why does `testCaldavConnection` instantiate `CalDAVClient` directly instead of going through `CalDavClientProvider` like the sync path does?" → This is noted as a scope-limit / sibling finding and folded into the Idiomatic fix; Minimal leaves it in place.

### Devil's advocate — if the reporter's theory is wrong

The reporter attributes the bug to `tsdav.fetchHomeUrl` not handling iCloud's `calendar-home-set` format — which is a plausible specific mechanism but may be too narrow. Three alternative causal sites worth considering: (1) **User-agent filtering by iCloud** — iCloud is known to return different response shapes to clients with "known" user-agents; tsdav does not set a user-agent that iCloud recognizes, and iCloud may be returning an HTML/error page instead of a proper DAV response (the TypeError on `.href` is consistent with this), in which case the fix is setting a user-agent header via the `headers` option in `createAccount`. (2) **HTTP/2 response quirks via Node's default agent** — iCloud upgrades to HTTP/2 aggressively, and tsdav's fetch implementation may mis-parse multiplexed responses; the reporter's direct-`curl` PROPFIND succeeding is consistent with curl defaulting to HTTP/1.1. (3) **`@lingui/core/macro` string-interpolation accidentally consuming error metadata** — unlikely but worth ruling out: `testCaldavConnection` uses `msg\`...\`` templating and the `error.message?.includes('CALDAV_SYNC_COLLECTION_NOT_SUPPORTED')` check; a prior tsdav version might throw an Error whose `.message` is a non-string, making `.includes` throw a TypeError that gets swallowed and re-thrown. Of these, (1) is the strongest counter-hypothesis and is worth *including* in the idiomatic fix — adding `User-Agent: 'TwentyCRM-CalDAV/1.0'` to `this.headers` costs ~1 LOC and eliminates a real unknown.

### Recommended next step

**proceed to fix**

### Open questions

- Which exact tsdav version (>= 2.1.5) first handles iCloud's `calendar-home-set` namespacing correctly? Requires running a test against `caldav.icloud.com` with a valid app-password to confirm. Reporter has a working credential — worth asking them to test a patched branch.
- Does iCloud's CalDAV endpoint require a specific `User-Agent` header (see Devil's advocate (1)) to respond with parseable DAV XML? If yes, the fix extends to `getBasicAuthHeaders` augmentation in `caldav.client.ts:69-72`.
- `logs.search` was not invoked because Phoenix's log backends do not cover a self-hosted Docker TwentyCRM deployment; is there a mechanism for reporter to upload relevant server logs so future triage can access runtime context?
- `parseCalDAVError` in `parse-caldav-error.util.ts` is not called from `testCaldavConnection`'s catch block — is this deliberate, or a bug? If intentional, the two error-translation layers should be reconciled.

<!-- MACHINE-READABLE FOOTER — DO NOT REMOVE; downstream skills parse this block -->
<!--phoenix:scout-summary
affected_repos: [ShankHarinath/twenty]
root_cause_confidence: 0.82
fix_direction_confidence: 0.72
confidence: 0.82
recommended_next_step: proceed
primary_causal_symbol: packages/twenty-server/src/modules/calendar/calendar-event-import-manager/drivers/caldav/lib/caldav.client.ts:97 (CalDAVClient.getAccount)
fix_shape_recommended: idiomatic
hypotheses_count: 4
hypotheses_confirmed: 1
converged: true
-->
