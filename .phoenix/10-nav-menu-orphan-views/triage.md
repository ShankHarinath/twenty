# Triage Report

## Issue
- **Repo:** ShankHarinath/twenty
- **Issue:** #10 — Favourites menu doesn't have all options in the UI and new views are not added and DB doesn't reflect changes
- **Reporter:** ShankHarinath  •  2026-04-21
- **Classification confidence (input):** 0.85
- **Scout triage rev:** 1

## Executive summary
Three symptoms reported — (1) DB has 7 favourites but UI shows only 5 (positions 2 and 6 hidden); (2) creating a new view on the already-favourited object does not make it appear; (3) deleting a favourite from the UI does not remove it from the DB — are **all explainable by a partially-migrated `navigationMenuItem` table combined with a frontend filter that silently drops rows whose referential integrity is broken**. The user's self-diagnosis is directionally correct: skipping the `upgrade:1-18:migrate-favorites-to-navigation-menu-items` command plus the `1-20:backfill-navigation-menu-item-type` command left a stale / partially-typed `navigationMenuItem` table, and the UI's filter (`filterAndSortNavigationMenuItems`) silently excludes rows that fail the type-specific integrity check. The "DB still has 7" that the user sees is almost certainly the deprecated legacy `favorite` table, which no writer touches any more (as of #18624 the frontend dual-write was removed); the UI reads `navigationMenuItem` exclusively, so writes/deletes go to a different table than the one the user is inspecting.

## Phase A — Repo & code mapping

### A.1 Search radius
- `list_repos` — 11 repos indexed; `twenty` is the relevant one (service-level repo; single-repo search scope).
- `group_list` — groups `backend`, `frontend`, `infra` exist but the symptom is entirely within the twenty monorepo (one codebase, no cross-service contract involved).
- **Chosen radius:** `twenty` only. No cross-repo investigation needed — the favourites flow lives entirely in `packages/twenty-server` (migration + GraphQL resolver) and `packages/twenty-front` (sidebar).

### A.2 Signals mapped to code
| Signal | File | Symbol |
|---|---|---|
| Favourites → navigation menu migration (v1.18) | `packages/twenty-server/src/database/commands/upgrade-version-command/1-18/1-18-migrate-favorites-to-navigation-menu-items.command.ts` | `MigrateFavoritesToNavigationMenuItemsCommand.runOnWorkspace` |
| Delete-orphan-favorites safety net (v1.18) | `packages/twenty-server/src/database/commands/upgrade-version-command/1-18/1-18-delete-orphan-favorites.command.ts` | `DeleteOrphanFavoritesCommand.runOnWorkspace` |
| Backfill `type` column for navigation menu items (v1.20) | `packages/twenty-server/src/database/commands/upgrade-version-command/1-20/1-20-backfill-navigation-menu-item-type.command.ts` | `BackfillNavigationMenuItemTypeCommand.backfillType` & `.cleanConflictingColumns` |
| Upgrade orchestration (version → command list) | `packages/twenty-server/src/database/commands/upgrade-version-command/upgrade.command.ts` | `UpgradeCommand` (commands_1180 at line 131; commands_1200 at line 154) |
| Feature flag key the migration toggles | `packages/twenty-shared/src/types/FeatureFlagKey.ts:17` | `IS_NAVIGATION_MENU_ITEM_ENABLED` |
| Sidebar read pipeline (the filter that silently drops rows) | `packages/twenty-front/src/modules/navigation-menu-item/common/utils/filterAndSortNavigationMenuItems.ts` | `filterAndSortNavigationMenuItems` |
| Sidebar rendering | `packages/twenty-front/src/modules/navigation-menu-item/display/sections/favorites/components/FavoritesSection.tsx` | `FavoritesSection` |
| Per-type metadata resolver (second drop point) | `packages/twenty-front/src/modules/navigation-menu-item/display/object/utils/getObjectMetadataForNavigationMenuItem.ts` | `getObjectMetadataForNavigationMenuItem` |
| Data hook that user-scopes items | `packages/twenty-front/src/modules/navigation-menu-item/display/hooks/useNavigationMenuItemsData.ts` | `useNavigationMenuItemsData` (filters by `userWorkspaceId`) |
| Create (add to favourite) | `packages/twenty-front/src/modules/navigation-menu-item/common/hooks/useCreateNavigationMenuItem.ts` | `useCreateNavigationMenuItem.createNavigationMenuItem` |

### A.3 Silent-filter mechanism (the UI drop)
`filterAndSortNavigationMenuItems` (lines 7-51) evaluates each row's `type`:
- `FOLDER` / `LINK` — always kept.
- `OBJECT` — requires `targetObjectMetadataId` to resolve in current `objectMetadataItems`.
- `VIEW` — requires `viewId` to exist in current `views` AND the view's `objectMetadataId` to resolve.
- `RECORD` — requires `targetRecordId`, `targetObjectMetadataId`, **and** a populated `targetRecordIdentifier`, plus object-metadata lookup.
Any row that fails its branch is silently discarded before `.sort((a,b) => a.position - b.position)`. Result: gaps in the displayed sequence exactly like the reported "positions 2 and 6 missing". A secondary, redundant dropper is in `useNavigationMenuItemSectionItems` (lines 64-77) which additionally requires `canReadObjectRecords` permission — if the affected users who see "only four" have slightly different roles, that is the mechanism.

### A.4 Affected repos
Only `twenty`. Fix will need:
- A new upgrade command (e.g. `1-20-reconcile-navigation-menu-item-referential-integrity.command.ts` or a one-shot repair CLI) that prunes / repairs rows whose `type`-specific foreign keys don't resolve, with telemetry.
- A frontend change so that silently-filtered rows surface as either a "stale favourite" indicator or a warning banner with a "clean up" affordance, rather than vanishing.
- Optional: verify the `useCreateNavigationMenuItem.createNavigationMenuItem` optimistic path (lines 47-133) is actually reflected on re-render when the user adds a view to an already-favourited object — the `relevantItems` filter on line 55 and `maxPosition` calc may collide with a stale row at the same `userWorkspaceId/folderId` scope.

## Phase B — Runtime evidence (kubectl)
**Could not be collected.** `kubectl --context kind-canonix get pods -A` returned `connection refused` on `127.0.0.1:59249`. The `kind-canonix` cluster is not running on this host. No production log corroboration was possible; triage relies on static analysis only. Recommend: start the local `kind-canonix` cluster (or point at the user's actual deployment) before the PR's test plan is exercised.

## Phase C — Commit history (relevant recent changes)
Most recent commits on the affected code paths (all on `main` / HEAD = `89300564ba`):

| SHA | Title | Why it matters |
|---|---|---|
| `f6beb06364` | Migrate favorites to navigation menu items (1.17 upgrade) — originally 1.17, renamed in #17991 | Introduced the migration command users who skipped see missing. |
| `2b702b2b45` | Navigation Menu Item - Migration in v1.18 (#17991) | Moved the migration into the 1.18 upgrade slot — **precisely the slot the user skipped.** |
| `ecb7270a7b` | fix: delete orphan favorites before 1.18 migration (#18219) | Companion migration — also only runs in 1.18. |
| `0ae62898a3` | fix: restructure navigationMenuItem type migration for safe upgrade path (#18722) | Prior `type` column migration assigned `VIEW` to every row, including folders/records/links — rows wrongly typed `VIEW` are exactly the rows `filterAndSortNavigationMenuItems` will drop. |
| `58336fb70f` | fix: navigation menu item type backfill and frontend loading (#18730) | Moved `BackfillNavigationMenuItemTypeCommand` from 1-19 → 1-20 upgrade path; added frontend cache-warming fix; **this is the hardening of the `type`-column recovery, and only runs on the 1.19→1.20 step**. |
| `6b48f197d4` | feat: deprecate WorkspaceFavorite in favor of NavigationMenuItem (#18624) | Removed `modules/favorites/` entirely from the frontend and killed the dual-write. Explains why user sees no change in the legacy `favorite` DB table after UI delete — the frontend no longer writes there. |
| `a121d00ddd` (HEAD) | feat: add color property to ObjectMetadata | Unrelated; confirms the above fixes are in HEAD. |

## Phase D — Similar past issues / PRs
- `gh issue list --repo ShankHarinath/twenty --search "favorites"` returned no hits — this fork has no prior issue history for the feature.
- Upstream `twentyhq/twenty` is outside the allowlist; I could not query it. The upstream fix PRs above (#18219, #18722, #18730) are the "fix pattern" that already exists in `main` of this fork — they just don't run for a workspace that jumped 1.17 → 1.19 because the 1.18 slot was skipped.

## Phase E — Root cause hypothesis

**Hypothesis.** The user's workspace was upgraded 1.17 → 1.19 skipping 1.18, so three distinct pieces of state are now inconsistent:

1. **Missing population.** `MigrateFavoritesToNavigationMenuItemsCommand` (the only writer that reads the legacy `favorite` table) never ran, so `navigationMenuItem` rows for that user may be a partial copy seeded by dev-seeder or by clicks pre-/post-migration. This explains "duplicate positioning in the DB, e.g. two objects at position 0" across users.
2. **Mistyped rows.** `BackfillNavigationMenuItemTypeCommand` (v1.20) wrote `type='VIEW'` as a default or left rows NULL (see PR #18722 description — "incorrectly assigned VIEW to all existing rows regardless of their actual type"). Rows whose true shape is OBJECT / RECORD but are now typed `VIEW` (with no `viewId` or with a `viewId` that doesn't resolve) are **silently filtered out by `filterAndSortNavigationMenuItems`** before ever reaching render. That is precisely why positions 2 and 6 disappear: they are still in the raw GraphQL response, fail the filter, and never appear in `navigationMenuItemsSorted`.
3. **Legacy-table divergence.** The user is inspecting the deprecated `core.favorite` table when they check "the DB". As of `6b48f197d4`, the frontend no longer writes to that table at all — `useCreateNavigationMenuItem` and `useDeleteNavigationMenuItem` target `navigationMenuItem`. So adding a new view's favourite writes to `navigationMenuItem` and is visible in the UI, while the legacy `favorite` table never changes, which the user perceives as "DB doesn't reflect changes". Symmetrically, removing "All Notes" from the UI updates `navigationMenuItem` (soft-delete or hard-delete) but the legacy `favorite` row persists — the exact observation the user reports.
4. **Apparent "new view not added" nuance.** The additional subtlety — "if I create a new view on the object in position 6 and favourite it, the UI list does not update" — is the second-order effect of #2: position 6 itself is a row whose `type` is wrong and is filtered out. When the user creates a view, `useCreateNavigationMenuItem.createNavigationMenuItem` computes `maxPosition` from `navigationMenuItems` (raw, unfiltered — line 62-65) and inserts at `maxPosition + 1`. That new row is typed `VIEW` with a valid `viewId` and *should* render; but if the `relevantItems` set the UI re-reads is stale-cache (the `minimalMetadata` cache-warm bug hinted at in #18730) or the user is checking a `folderId` scope whose parent folder is itself filtered out, the new row won't surface until a page reload.

**Confidence: 0.78** — Strong static-analysis evidence and three upstream fix PRs that match the exact phenomena; blocked at ≥ 0.85 because (a) I could not confirm with runtime logs (cluster unavailable) and (b) the user hasn't shared actual row contents (`type`, `viewId`, `targetObjectMetadataId`, `folderId`) so we're inferring the mistyping pattern rather than observing it.

## Fix direction (for Architect)
1. **Primary fix (server).** Add a new `1-20:reconcile-navigation-menu-item-referential-integrity` command that:
   - For each row in `core.navigationMenuItem`, re-asserts its `type` based on populated FK columns (mirror `BackfillNavigationMenuItemTypeCommand.backfillType`), then verifies the FKs resolve in the current workspace metadata. Rows that still fail are logged and soft-deleted.
   - Runs idempotently on every `upgrade` invocation so workspaces that skipped 1.18 converge when the next upgrade is run.
2. **Safety-net (server).** Also run `MigrateFavoritesToNavigationMenuItemsCommand` conditionally when the legacy `favorite` table still contains rows for users whose `navigationMenuItem` set is empty/smaller — currently the command early-returns if `IS_NAVIGATION_MENU_ITEM_ENABLED` is on (line 83-89), which is exactly the trap a 1.17 → 1.19 upgrade falls into (the flag was flipped by some other path, migration skipped).
3. **UI observability (frontend).** Change `filterAndSortNavigationMenuItems` from silent drop to *annotate-and-render-disabled* for rows failing the integrity check, plus surface a small "clean up orphaned favourites" banner when any row is annotated. This is the visibility bug that let the issue go undetected across users.
4. **Docs.** Update the upgrade guide to explicitly warn that skipping a minor (e.g. 1.17 → 1.19) leaves `navigationMenuItem` in a broken state; suggest users run `nx run twenty-server:command -- upgrade` with every prior version explicitly.

## Open questions / blockers
- Runtime log corroboration impossible: `kind-canonix` cluster unreachable (`connection refused`). Before merging, engineer should confirm on a live workspace that `type='VIEW'` rows with no resolving `viewId` are indeed the hidden positions.
- Need actual row dump from the reporter's workspace: `SELECT id, type, viewId, targetObjectMetadataId, targetRecordId, link, folderId, position, "userWorkspaceId" FROM core."navigationMenuItem" WHERE "workspaceId" = '<ws>' ORDER BY "userWorkspaceId", position;` — this confirms or refutes the mistyping theory in one query.
- Is the user self-hosting a fork at exactly this repo's HEAD, or at a tagged release behind HEAD? The `1-20-backfill-navigation-menu-item-type` command exists at HEAD but may not be present on the release they are running; that materially changes whether fix (1) above is a *new* command or a *run-the-existing* recovery.

## Recommended next step
Architect should plan a server-side reconciliation command (bullet 1 above) plus the frontend annotation change (bullet 3). A single PR against `twenty` covers both; no cross-repo coordination needed.

---

```yaml
# machine-readable footer
affected_repos:
  - ShankHarinath/twenty
confidence: 0.78
recommended_next_step: architect_plan
blockers:
  - kubectl_context_unreachable: kind-canonix at 127.0.0.1:59249
  - no_runtime_log_corroboration
  - need_row_dump_from_reporter_db
primary_suspects:
  - packages/twenty-front/src/modules/navigation-menu-item/common/utils/filterAndSortNavigationMenuItems.ts:filterAndSortNavigationMenuItems
  - packages/twenty-server/src/database/commands/upgrade-version-command/1-18/1-18-migrate-favorites-to-navigation-menu-items.command.ts:MigrateFavoritesToNavigationMenuItemsCommand.runOnWorkspace
  - packages/twenty-server/src/database/commands/upgrade-version-command/1-20/1-20-backfill-navigation-menu-item-type.command.ts:BackfillNavigationMenuItemTypeCommand
  - packages/twenty-front/src/modules/navigation-menu-item/common/hooks/useCreateNavigationMenuItem.ts:useCreateNavigationMenuItem
related_prs_in_history:
  - "#17991 Navigation Menu Item - Migration in v1.18"
  - "#18219 delete orphan favorites before 1.18 migration"
  - "#18624 deprecate WorkspaceFavorite in favor of NavigationMenuItem"
  - "#18722 restructure navigationMenuItem type migration"
  - "#18730 navigation menu item type backfill and frontend loading"
```
