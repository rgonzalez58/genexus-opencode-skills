---
name: slo-non-gam-debugging
description: Diagnosing SLO chains that include non-GAM clients — URL algorithm, auto-redirect, session state
---

# Debugging SLO with Non-GAM Clients
Scope: when the SLO chain involves clients that do NOT have GAM applied (they use HTTPClient for OAuth), the debugging approach changes significantly because these clients produce NO GAM traces

Related files:
- [GAM Single Log Out (SLO)](domain-slo.md) — 3-phase SLO flow, session hierarchy, Non-GAM Client SLO
- [SLO External IDP Analysis Playbook](domain-slo-external-idp.md) — external IDP chaining
- [GAM-as-SP — OAuth 2.0 Common Flow (observable in trace)](../authentication/external-providers/oauth20/common/oauth20-common-flow.md) — shared `state` / callback machinery and trace format
- [GAM Debugging Core](../debugging/common-debugging.md) — trace activation
- [GAM SQL Diagnostic Queries](../debugging/sql-diagnostic-queries.md) — daughter session queries
- [GAM Behavioral Patterns](../debugging/common-behavioral-patterns.md) — URL selection algorithm
- [GAM Backoffice — Applications (GAM_Applications)](../backoffice/applications.md) — `ClientSingleLogoutCustomURLsSLO` config

---

## Symptom
SLO triggered from IDP → sub-IDP processes SLO → chain breaks silently → daughter sessions on non-GAM clients remain active

## Step 1 — Confirm Non-GAM Path Was Taken
In the sub-IDP log, search for:

```
grep -i "isCustomURLSLO" sub-idp.log
```

If you see `isCustomURLSLO:true` — GAM took the non-GAM client path. This means:
- GAM did NOT append `oauth/gam/callback` to the URL
- GAM did NOT send `server_ip=0` or `repository` parameters
- GAM sent only: `client_id`, `redirect_uri`, `state`, `token`

## Step 2 — Verify Which Custom URL Was Selected
Search for the URL selection trace:

```
grep -i "AppCliSLOURL\|ClientSingleLogoutCustomURLsSLO" sub-idp.log
```

Critical check: does `AppCliSLOURL` match the actual deployment URL of the non-GAM client?

The `GAMRemotLogoutDetectAppSLOURL` algorithm does progressive path-stripping:
- First tries exact app-path match (most specific)
- Falls to domain-level match (least specific — picks FIRST domain match)

Common failure: multiple URLs share the same domain → algorithm picks the wrong one (first match wins)

## Step 3 — Verify Non-GAM Client Received the Redirect
In the non-GAM client log:
- ZERO SLO traces — redirect never arrived (wrong URL, 404, DNS failure)
- SLO traces present but no redirect back — handler did not auto-redirect (manual button click required)

Since non-GAM clients produce no GAM traces, check:
- Web server access logs (IIS / Kestrel / Tomcat) for the SLO URL hit
- Application-level logging if the SLO handler has any

## Step 4 — Verify the Handler Redirects Back
The non-GAM SLO handler MUST auto-redirect to `redirect_uri?state=<state>` for the chain to continue

Test independently:

```bash
curl -v "http://app-d.com/MyApp/slo?client_id=xxx&redirect_uri=http://client-a.com/oauth/gam/callback&state=SLOInt123&token=abc"
```

- Expected: HTTP 302 to `redirect_uri?state=SLOInt123`
- Failure: HTTP 200 (page rendered, waiting for user click) or HTTP 404

## Step 5 — SQL Diagnosis — Daughter Session State
See [GAM SQL Diagnostic Queries](../debugging/sql-diagnostic-queries.md) "Daughter sessions" query

Interpretation:
- All daughters with `SesEndDate = NULL` — SLO never started processing daughters
- Some daughters with `SesEndDate` set — SLO chain broke at a specific daughter

## Checklist
- Sub-IDP log shows `isCustomURLSLO:true`
- `AppCliSLOURL` matches actual deployment URL
- `ClientSingleLogoutCustomURLsSLO` in Backoffice has correct URLs
- No duplicate domain entries that could cause wrong selection
- Non-GAM handler is accessible (no 404/500)
- Non-GAM handler auto-redirects (no manual steps)
- Redirect URL format is `redirect_uri?state=<state>` (not `?&state=`)
- SQL shows daughter sessions marked (`SesSts='F'`)
- SQL shows `SesEndDate` progression (which daughters were notified)

---

## Silent-Failure Catalog (Non-GAM Focus)
- Condition: non-GAM custom SLO URL misordered
	* Trace signature: `GAMRemotLogoutDetectAppSLOURL` selects a domain-level match that is not the active deployment
	* Symptom: signout reaches the wrong endpoint (404 or wrong deployment)
	* Root cause: `ClientSingleLogoutCustomURLsSLO` has multiple entries sharing a domain and the wrong one is first in the list
	* Fix: reorder `ClientSingleLogoutCustomURLsSLO` so the correct URL appears first

- Condition: non-GAM handler does not auto-redirect
	* Trace signature: no callback return after the signout was sent to the non-GAM client
	* Symptom: SLO chain stops; downstream daughters never notified
	* Root cause: handler renders a page and waits for user click instead of emitting HTTP 302
	* Fix: emit an immediate 302 to `redirect_uri?state=<state>`; never gate the redirect behind user interaction

- Condition: state snapshot not found in `LoginTmpTokenState`
	* Trace signature: callback reaches `Otherwise Parm_State: SLO…` but the snapshot load returns empty
	* Symptom: user silently redirected to login instead of `FromURL`
	* Root cause: TTL expired or the upstream INSERT failed
	* Fix: check DB logs for `LoginTmpTokenState` insert failures; confirm TTL exceeds expected SLO round-trip time

- Condition: `state` missing on the return URL from the non-GAM handler
	* Trace signature: callback at the sub-IDP has no `state` query parameter
	* Symptom: state machinery cannot resume; sub-IDP does not know which daughter to notify next
	* Root cause: the non-GAM handler built the redirect without echoing `state`
	* Fix: use `Format(!"%1?state=%2", &redirect_uri, &state)` in the handler; do NOT drop or rename `state`

- Condition: `GAMRemoteLogoutBehavior = ClientOnly`
	* Trace signature: no daughter loop on the sub-IDP; only the requesting client is finished
	* Symptom: non-GAM daughters never receive a signout
	* Root cause: repository configured to terminate only the requesting client
	* Fix: change `GAMRemoteLogoutBehavior` to `All` in Backoffice → Repository
