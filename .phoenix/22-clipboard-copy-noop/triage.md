# Triage Report — ShankHarinath/twenty#22

## Summary
Copy buttons across the Twenty frontend (API key display, Workspace invite link in Settings > Members, and onboarding InviteTeam link) all delegate to a single hook `useCopyToClipboard` at `packages/twenty-front/src/hooks/useCopyToClipboard.tsx`. The hook calls `await navigator.clipboard.writeText(valueAsString)` inside a `try { … } catch { … }` block. When the async-Clipboard API is unavailable or write silently resolves without actually writing (self-hosted HTTP deployments, Firefox non-secure contexts, clipboard permission revoked, document-not-focused edge cases), the code path still reaches `enqueueSuccessSnackBar(...)` and the user sees "Copied to clipboard" — but the clipboard is empty. There is no secure-context guard, no `document.execCommand('copy')` fallback, and no post-write verification.

The reporter's environment (self-hosted Docker, cloud-hosted Enterprise Linux server, Firefox client, "docker 2+") is exactly the population most likely to hit this: non-HTTPS origin where `navigator.clipboard` is either `undefined` or restricted in Firefox. Their note that "Previously other fields with copy button also didn't work" confirms the root is in the shared hook, not in any single callsite.

## Affected repositories
- `ShankHarinath/twenty` — the fix is a pure frontend change to `packages/twenty-front/src/hooks/useCopyToClipboard.tsx`.

No other indexed repo (backend, erpnext, etc.) participates in this bug. No API contract changes.

## Evidence

### Phase A — Code map
- `mcp__gitnexus__query "copy to clipboard button"` surfaced the canonical implementation at `packages/twenty-front/src/hooks/useCopyToClipboard.tsx:5-33`.
- `mcp__gitnexus__impact target=useCopyToClipboard direction=upstream` returned LOW graph-level upstream (the hook is used via pattern hooks returning a callback, so static analysis under-counts); textual search shows **~30 callsites** across the frontend:
  - `packages/twenty-front/src/modules/settings/developers/components/ApiKeyInput.tsx:36` — the "API key" button in the issue.
  - `packages/twenty-front/src/modules/workspace/components/WorkspaceInviteLink.tsx:47` — the "Settings > Members > Invite User" copy button (confirmed used in `packages/twenty-front/src/pages/settings/members/SettingsWorkspaceMembers.tsx:264`).
  - `packages/twenty-front/src/pages/onboarding/InviteTeam.tsx:119` — onboarding copy.
  - Plus ~26 other callsites: `LightCopyIconButton`, `EmailsFieldInput/Display`, `PhonesFieldDisplay`, `LinksFieldDisplay`, `JsonFieldDisplay`, `SettingsSSOOIDCForm`, `SettingsSSOSAMLForm`, `WorkflowEditTriggerWebhookForm`, `SettingsTwoFactorAuthenticationMethod`, `SignInUpTwoFactorAuthenticationProvision`, `SettingsAdminJsonDataIndicatorHealthStatus`, `ObjectOptionsDropdownVisibilityContent`, `ToolStepRenderer`, `RoutingDebugDisplay`, `CodeExecutionDisplay`, `TerminalOutput`, `ThinkingStepsDisplay`, etc. **All** paths funnel through this one hook — so a fix here is a blanket fix.
- The hook body (verbatim from `packages/twenty-front/src/hooks/useCopyToClipboard.tsx:11-31`):

```tsx
const copyToClipboard = async (valueAsString: string, message?: string) => {
  try {
    await navigator.clipboard.writeText(valueAsString);
    enqueueSuccessSnackBar({
      message: message || t`Copied to clipboard`,
      ...
    });
  } catch {
    enqueueErrorSnackBar({ message: t`Couldn't copy to clipboard`, ... });
  }
};
```

No `isSecureContext` check, no `execCommand` fallback, no retry, no focus check.

### Phase B — Runtime signal (kubectl)
Cluster `kind-canonix` is not reachable from the investigation host (connection refused on the API server port). Skipped — this is a pure frontend/browser runtime bug anyway; server logs do not carry evidence of clipboard-write failures because they occur entirely in the browser.

### Phase C — Recent commits on the affected file
Full history of `packages/twenty-front/src/hooks/useCopyToClipboard.tsx`:
- `316f2ec38c` (2025-07-30) — *refactor: to useCopyToClipboard to catch errors - when user has disable copy clipboard permission in browser (#13330)* — wrapped the write in try/catch and added the optional `message` parameter. This is the last functional change; it did not introduce the bug but it is the change that **masked** the silent-success failure by assuming any unavailability would throw (which is only true when `navigator.clipboard` is undefined, not when the write resolves without effect).
- `288f0919db` (prior) — snackbar refactor.
- `9353e777ea` — initial "Copy JSON values on click".

No fallback path (`document.execCommand`, hidden `<textarea>` + `select()`) has ever been present in this repo.

### Phase D — Similar past issues and PRs
- `gh issue list --repo ShankHarinath/twenty --search "copy clipboard"` returned only issue #22 itself — the fork has no prior clipboard reports.
- `gh pr list --repo ShankHarinath/twenty --search "copy clipboard"` returned zero results.
- Upstream is denied by the counterfactual gate; see `leak_signals` in the footer for any hints surfaced.

### Phase E — Root-cause hypothesis
The `useCopyToClipboard` hook is the single point of failure. Its `navigator.clipboard.writeText` path succeeds and reaches `enqueueSuccessSnackBar` whenever the promise resolves — but there are real browser conditions (self-hosted HTTP origin, Firefox non-secure context, document-not-focused, permission policy without explicit throw) where the promise either never throws or resolves without actually writing. Because there is no fallback and no verification, the user is told "Copied to clipboard" while the clipboard is empty — exactly matching the reporter's bug.

Recommended fix shape (for Architect):
1. Gate on `navigator?.clipboard && window.isSecureContext`; if unmet, use a fallback (`document.execCommand('copy')` on a hidden `<textarea>` with `select()`), then show success only if that returns `true`.
2. Keep the async path but verify it resolved (promise resolution is the current proxy); additionally ensure the caller's button `onClick` does not await inside a `setTimeout`/focus-losing wrapper.
3. Show an **error** snackbar when no path succeeded, rather than defaulting to success.
4. (Optional) Expose the fallback via a shared helper so the 30+ callsites inherit it automatically.

Risk of change: LOW — localized to `useCopyToClipboard.tsx`, ~30 callers, all use the hook's returned `copyToClipboard` function only. Signature-compatible fix keeps `(value: string, message?: string) => Promise<void>`. No impact on backend, GraphQL, or schema.

## Open questions
- Does the reporter's deployment serve Twenty over HTTPS or plain HTTP? The environment line ("cloud hosted on Enterprise Linux … docker 2+") does not say. A quick confirmation narrows the root cause from "missing fallback" to "broken in non-secure context specifically."
- Does the issue reproduce in Chrome on the same server? The reporter only tested Firefox; Firefox is stricter about clipboard on HTTP than Chrome, which would point squarely at the secure-context gap.
- Have you set up Twenty to run in an `<iframe>` or behind a reverse proxy that strips focus/user-activation? Both can cause `writeText` to resolve without writing.

## Recommended next step
**Implement fix in `ShankHarinath/twenty`**. Scope is a single hook file (`packages/twenty-front/src/hooks/useCopyToClipboard.tsx`) plus an optional small shared helper. No schema, backend, or cross-repo work needed. Add a regression test mocking `navigator.clipboard = undefined` and asserting the fallback path completes + returns success (and that true failure produces the error snackbar).

---

```yaml
# phoenix-triage-footer
schema: phoenix-triage/v1
issue:
  repo: ShankHarinath/twenty
  number: 22
rev: 1
confidence: 0.78
classification: bug-frontend-clipboard
affected_repos:
  - ShankHarinath/twenty
searched_repos:
  - ShankHarinath/twenty
recommended_next_step: implement-fix
blockers:
  - kubectl_cluster_unreachable   # kind-canonix API server refused connection; not needed for this frontend bug
  - no_prior_issues_or_prs_in_fork
leak_signals:
  - "upstream PR hint observed in commit 316f2ec38c body: twentyhq/twenty/issues/13292 (discussion) — not used inline per counterfactual gate"
primary_suspects:
  - path: packages/twenty-front/src/hooks/useCopyToClipboard.tsx
    symbol: useCopyToClipboard
    lines: "5-33"
    reason: "Single chokepoint for all copy buttons; navigator.clipboard.writeText with no fallback, no secure-context guard, success snackbar shown on any non-throwing resolution."
secondary_touchpoints:
  - packages/twenty-front/src/modules/settings/developers/components/ApiKeyInput.tsx
  - packages/twenty-front/src/modules/workspace/components/WorkspaceInviteLink.tsx
  - packages/twenty-front/src/pages/onboarding/InviteTeam.tsx
```
