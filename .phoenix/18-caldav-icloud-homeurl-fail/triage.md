# Triage Report — Issue #18 (ShankHarinath/twenty)

**Title:** CalDAV sync fails with iCloud: "No calendar with event support found" / "cannot find homeUrl"
**Reporter:** ShankHarinath
**Rev:** 1
**Classification confidence (input):** 0.95
**Scout confidence (root cause):** 0.72

## Summary

The CalDAV connect flow (`ImapSmtpCaldavService.testCaldavConnection`) calls `CalDAVClient.listCalendars()` then `CalDAVClient.validateSyncCollectionSupport()`. Both paths invoke tsdav's `createAccount({ accountType: 'caldav' })` followed by `fetchCalendars({ account })`. Against `caldav.icloud.com`, tsdav's internal principal/home-URL discovery (`fetchHomeUrl`) fails — raising either `cannot find homeUrl`, `Cannot read properties of undefined (reading 'href')` (a property access on an undefined DAV element inside tsdav's XML parser), or — if the home URL is resolved but no calendars carry `VEVENT` in `components` — the guard added in commit `843fda7564` throws `No calendar with event support found`.

All three symptoms converge on one code region: `packages/twenty-server/src/modules/calendar/calendar-event-import-manager/drivers/caldav/lib/caldav.client.ts` (the `getAccount` / `listCalendars` / `validateSyncCollectionSupport` triad) plus possibly a patch to `tsdav` via `packages/twenty-server/patches/`.

## Phase A — Code mapping

Searched repo: `twenty` only (the home repo). No group contains `twenty`; error signatures are TypeScript + NestJS server-side and have no cross-repo surface (error strings originate in `tsdav` → surface in twenty's own controller layer).

Signals resolved to code:

| Signal | File / Symbol | Confidence |
|---|---|---|
| `CALDAV connection failed: …` | `packages/twenty-server/src/engine/core-modules/imap-smtp-caldav-connection/services/imap-smtp-caldav-connection.service.ts:135-154` (`ImapSmtpCaldavService.testCaldavConnection`) | 1.0 |
| `No calendar with event support found` | `packages/twenty-server/src/modules/calendar/calendar-event-import-manager/drivers/caldav/lib/caldav.client.ts:160-162` (`CalDAVClient.validateSyncCollectionSupport`) | 1.0 |
| `cannot find homeUrl` | mapped to `CalendarEventImportDriverExceptionCode.NOT_FOUND` in `parse-caldav-error.util.ts:30-34`; raised inside `tsdav` (`fetchHomeUrl`) | 0.95 |
| `Cannot read properties of undefined (reading 'href')` | raised inside `tsdav`'s XML deserializer when `calendar-home-set` response lacks expected `href` structure; surfaces through `createAccount` or `fetchCalendars` in `CalDAVClient.getAccount` / `listCalendars` | 0.85 |
| `fetchHomeUrl` | external (`tsdav@^2.1.5`, `packages/twenty-server/package.json:179`); called transitively by `createAccount({ accountType: 'caldav' })` | 1.0 |
| `calendar-home-set` / PROPFIND | internal tsdav DAV discovery — no twenty-side code handles it directly | 0.9 |

Impact of `testCaldavConnection` (downstream): called only by `ImapSmtpCaldavService.testImapSmtpCaldav`, which is the resolver/controller entry for the Add-Account flow (IMAP/SMTP/CALDAV switch). Blast radius is contained to the CalDAV connect path.

Impact of `CalDAVClient.validateSyncCollectionSupport` (upstream): single caller is `testCaldavConnection`. `CalDAVClient.listCalendars` is additionally used by `CalDavGetEventsService.getEvents` (the periodic sync), meaning any fix in `getAccount` affects ongoing sync as well — desired.

## Phase B — Runtime evidence

**Blocked:** kubectl context `kind-canonix` is unreachable (`connection refused` on 127.0.0.1:59249). Cannot corroborate the reported symptom with live pod logs. Proceeding on code + reporter evidence only. This reduces Scout confidence by ~0.1.

## Phase C — Recent commits on affected files

| SHA | Subject | Relevance |
|---|---|---|
| `843fda7564` (2026-01-22) | CalDav throw error if a user tries to connect an unsupported server. (#17363) | Introduced `validateSyncCollectionSupport`, the exact source of `No calendar with event support found`. This guard was intended to reject legacy servers that don't support RFC 6578 sync, but it fires for iCloud when the prior `fetchCalendars` misparses and returns calendars without `components: ['VEVENT']` populated — or when `fetchCalendars` returns empty because the home URL resolution produced a bad root. |
| `5f70d388ef` (2025-12-04) | Fix caldav issues (#16297) | Small touch-ups to `caldav.client.ts` and the connection service; unrelated to home-URL discovery. |
| `3e8fa3120d` | feat: CalDav Driver (#13170) | Original driver landing. |

No commit in the last 30 days modifies `caldav/lib/caldav.client.ts`; the most likely proximate cause is the January 2026 addition of the `validateSyncCollectionSupport` guard combined with tsdav's longstanding iCloud-parse quirk, not a fresh regression.

## Phase D — Similar past issues and PRs

Only the home repo `ShankHarinath/twenty` was searched (counterfactual constraint). Issue #18 is the sole CalDAV-related issue; no prior fix PR targets iCloud specifically. Historic PRs (#16297, #17363) touched the same driver but for different symptoms.

## Phase E — Root-cause hypothesis

The three reported error messages are three stages of the same failure mode:

1. **Primary cause:** `CalDAVClient.getAccount()` (line 97-109) calls tsdav's `createAccount` with just `serverUrl`, `accountType: 'caldav'`, and credentials. tsdav internally runs principal discovery → `fetchHomeUrl` against iCloud. iCloud's `calendar-home-set` PROPFIND response is accepted by some tsdav versions but version `^2.1.5` mis-locates the `href` element, producing either `Cannot read properties of undefined (reading 'href')` (best-case: response shape is unexpected) or `cannot find homeUrl` (worst-case: href value is empty).

2. **Secondary failure mode:** When discovery partially succeeds but the resolved home URL points at a container whose child calendars have an empty or unparsed `components` array, `fetchCalendars` returns calendars, but `validateSyncCollectionSupport` (line 156-162) finds no calendar with `VEVENT` in `components` and throws `No calendar with event support found`.

3. **Input-shape contribution:** The reporter experimented with both `https://caldav.icloud.com/` and `https://caldav.icloud.com/<DSID>/principal/` — tsdav's discovery flow expects a root URL it can redirect/follow from. The principal-URL input bypasses part of the auto-discovery chain tsdav performs and changes which of the three errors surfaces.

**Recommended fix surface (for Architect):**
- Primary: harden `CalDAVClient.getAccount()` to perform an iCloud-specific PROPFIND to discover the principal/home URL when `serverUrl` host is `*.icloud.com`, OR wrap `createAccount` with a more resilient fallback that issues the principal PROPFIND directly and passes `rootUrl` + `principalUrl` + `homeUrl` explicitly to bypass tsdav's auto-discovery.
- Secondary: translate these three low-level messages into a single user-friendly `iCloud configuration` error in `testCaldavConnection` and in `parse-caldav-error.util.ts` (currently only `cannot find homeUrl` is mapped; `No calendar with event support found` and the TypeError fall through to `UNKNOWN`).
- Optional: add a tsdav patch in `packages/twenty-server/patches/` (the patches dir exists but currently has no `tsdav` patch) if the iCloud parse bug is confirmed in `tsdav@2.1.5`.

**Confidence rationale (0.72):**
- +0.85 from a clean symbol-to-error mapping (three error strings all trace to explicit code locations)
- +0.10 from a plausible, well-documented external interaction (tsdav vs. iCloud CalDAV peculiarities)
- −0.10 for missing runtime log confirmation (kubectl unreachable)
- −0.15 for two viable sub-hypotheses (tsdav parse bug vs. iCloud-specific discovery inputs) that a fix needs to disambiguate through a reproduction test

## Affected repositories

Only `ShankHarinath/twenty` requires code changes. No cross-repo surface.

## Open questions

1. Which of the three symptoms is the *first* one thrown in a fresh repro? (Answer would tell us whether the fix is at `createAccount`, at `validateSyncCollectionSupport`, or at `parse-caldav-error.util.ts`.)
2. Is the tsdav version (`^2.1.5`) still current? Upstream tsdav has had iCloud-related fixes; a dependency bump may resolve this without twenty-side logic changes.
3. Is the reporter using the full DSID URL or the bare host? The issue text says both were tried, but logs from a single reproduction run would clarify which entry-point error chain fires.
4. Runtime confirmation is pending — kubectl context `kind-canonix` was unreachable during triage. Architect should either (a) repair kubectl access and re-pull logs, or (b) run a local repro with a test iCloud account.

## Recommended next step

Architect to design a fix targeting:
- `packages/twenty-server/src/modules/calendar/calendar-event-import-manager/drivers/caldav/lib/caldav.client.ts` — add explicit principal/home-URL resolution before calling `createAccount`, with a provider-specific branch for `*.icloud.com`.
- `packages/twenty-server/src/modules/calendar/calendar-event-import-manager/drivers/caldav/utils/parse-caldav-error.util.ts` — map `No calendar with event support found` and the TypeErrors to a clearer `NOT_FOUND` / `PROVIDER_MISCONFIGURED` code.
- `packages/twenty-server/src/engine/core-modules/imap-smtp-caldav-connection/services/imap-smtp-caldav-connection.service.ts:testCaldavConnection` — surface an iCloud-specific `userFriendlyMessage` when host matches `*.icloud.com` and discovery fails.

Consider adding a regression test under `packages/twenty-server/src/modules/calendar/calendar-event-import-manager/drivers/caldav/` that stubs an iCloud-shaped PROPFIND response.

---

<!--phoenix:scout-summary
affected_repos:
  - ShankHarinath/twenty
confidence: 0.72
recommended_next_step: proceed
leak_signals: []
-->
