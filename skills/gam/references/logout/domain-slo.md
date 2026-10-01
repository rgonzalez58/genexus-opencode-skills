---
name: domain-slo
description: Single Log Out — GAMRemote flow, session hierarchy, LogoutType, external IDP chaining, non-GAM clients
---

# GAM Single Log Out (SLO)
## DEFINITION
Single Log Out (SLO) is the mechanism by which logging out from one application triggers session termination across all applications connected through GAM. In a GAMRemote architecture, SLO coordinates logout between Client Applications and Identity Providers (IDPs) using browser redirects and state management

---

## BEHAVIOR
### The SLO Architecture (GAMRemote)
An SLO cycle involves a Client Application and an Identity Provider (IDP). Each GAM instance has a role determined by the `server_ip` parameter (observable in traces):

- Client
	* Trace marker: `Parm_isServerIP:0`
	* Description: The application where the user initiated logout
- IDP
	* Trace marker: `Parm_isServerIP:1`
	* Description: The Identity Provider that manages centralized sessions
- Sub-IDP (chained)
	* Trace marker: `Parm_isServerIP:2`
	* Description: A node that is both a client (of a parent IDP) and an IDP (for its own children)

### Client-Side Initiation (Client role)
When a user logs out from a GAMRemote client:
- GAM saves the current context (state snapshot) to the database for later recovery
- GAM marks the local session as Finished
- GAM performs a 302 Redirect to the IDP's `/oauth/gam/signout` endpoint
	* Parameters sent: `client_id`, `redirect_uri` (client's post-logout URL), `server_ip=0`, `state`, and `token` (the session's external token)

### Logout_internal — SLO Orchestrator Entry Point
`Logout_internal` is the orchestrator procedure that drives SLO end-to-end on each node. Signature (observable in traces):

```
Logout_internal(CacheRepository, isForceLogout, SessionType, FinishSession, GAMState, &RedirURL, &isOK, &Errors)
```

The orchestrator coordinates, in order:
- `GAMRemoteLogout` — kills IDP-side sessions (parent + daughters as configured)
- `GAMExternalAuthenticationSignout` — builds the external IDP redirect URL when the node authenticated against an external IDP
- `GAMStateClientAPI` — serializes the context snapshot to the `LoginTmpTokenState` DB table
- Returns `&RedirURL` (the next browser redirect) to the HTTP handler so the browser can carry the chain forward

Trace signature (v18u15+):

```
genexus.security.api.Logout_internal - Start_Method - {"data":{…}}
genexus.security.api.Logout_internal - End_Method - {"data":{…}}
```

Use `Logout_internal` as the anchor when reading a trace: everything that happens between its `Start_Method` and `End_Method` belongs to one SLO hop

### IDP-Side Processing (IDP role)
When the IDP receives the `/oauth/gam/signout` request:
- GAM extracts the incoming token and looks up the corresponding session
- From the session record, GAM identifies the parent session (the IDP-side session)
- GAM finishes the specific child session in the IDP database
- GAM evaluates the GAMRemoteLogoutBehavior property of the repository:
	* ClientOnly: Does NOT finish the parent session. User stays logged in at the IDP
	* ClientAndIP or All: Finishes the parent session AND destroys the IDP browser session (cookie)
- If subscriptions to `GAMEvents.Repository_Logout` exist, they are triggered
- If the behavior is All, GAM recursively signs out all other connected clients (daughter sessions)
- GAM redirects back to the Client using the `redirect_uri` with the original `state`

### Client-Side Completion (Return from IDP)
- The Client receives the redirect back from the IDP
- GAM processes the `state` parameter to recover the context
- The Client displays the login screen or the custom `ValidURLAfterSLO`

---

## SLO with External IDP, Sub-IDP Escalation, and Daughter Notification
Architecture (state snapshot mechanism, 4-step external-IDP flow, sub-IDP escalation state machine, two-phase daughter notification) and diagnostic playbook (4-step trace verification, Silent-Failure Catalog): [SLO External IDP Analysis Playbook](domain-slo-external-idp.md)

---

## TRACE SIGNATURES
### Client-Side Logs
- `GAMTrace-SLOProcess - Client - Parm_isServerIP:0`
	* Meaning: This node acts as the SLO client
	* Absence indicates: SLO not initiated or wrong entry point
- `GAMTrace-SLOProcess - Client - External token: <TOKEN>`
	* Meaning: Client holds this IDP token
	* Absence indicates: Token lookup failed
- `GAMTrace-SLOProcess - Local token: <TOKEN> UserGUID: <USER> AppId: <APP>`
	* Meaning: Local session identified for termination
	* Absence indicates: Session lookup failed
- `GAMTrace-SLO Redirect URL: <IDP_URL>/oauth/gam/signout?client_id=…`
	* Meaning: Outbound redirect to IDP for SLO
	* Absence indicates: Redirect not built — check GAMRemoteLogoutBehavior

### IDP-Side Logs
- `GAMTrace-==+++GAMExternalAuthenticationGAMRemote ====== START ---`
	* Meaning: SLO entry point reached
	* Absence indicates: Request not reaching GAM
- `GAMTrace-SLOProcess - Parm_isServerIP:1`
	* Meaning: This node acts as the IDP
	* Absence indicates: Wrong role assignment
- `GAMTrace-SLOProcess - IDP - &TokenToFinish: <TOKEN>`
	* Meaning: Token received from Client
	* Absence indicates: Token extraction failed
- `GAMTrace-SLOProcess - IDP - SLO process initiated in Client - &ParentToken: <TOKEN>`
	* Meaning: Parent session identified
	* Absence indicates: Session lookup failed — child session not found
- `GAMTrace-SLOProcess - IDP - Finish parent token: <TOKEN>`
	* Meaning: Parent session terminated
	* Absence indicates: GAMRemoteLogoutBehavior is ClientOnly

### Sub-IDP Escalation Logs
- `GAMTrace-SLOProcess - Parm_isServerIP:0   Parm_FirstCall:0`
	* Meaning: Initial state (client role)
- `GAMTrace-SLOProcess - IDP-IDP`
	* Meaning: Escalation detected — switching to sub-IDP
- `GAMTrace-SLOProcess - Parm_isServerIP:2   Parm_FirstCall:1`
	* Meaning: After escalation (sub-IDP role)

### External IDP SLO Logs
- `GAMTrace-GAMExternalAuthenticationSignout`
	* Meaning: GAM is calling the external IDP's signout
- `GAMTrace-Oauth20 Redirect Signout URL:`
	* Meaning: The full URL sent to the external IDP
- `GAMTrace-GAMExternalAuthenticationInputLoadParam: &OriginType:` (empty)
	* Meaning: Return from external IDP — expected empty
- `GAMTrace-GAMExternalAuthenticationInputValidParam - Otherwise Parm_State: SLO…`
	* Meaning: State fallback triggered (restoring snapshot)
- `GAMTrace-GAMExternalAuthenticationInputValidParam-&APIStateClientPar:`
	* Meaning: Restored snapshot content
- `GAMTrace-GAMExternalAuthenticationInput &RedirToURL OK:`
	* Meaning: Final redirect to client

### Custom SLO URL Detection Logs
- `GAMTrace-==+++GAMRemotLogoutDetectAppSLOURL ====== START ---`
	* Meaning: Custom URL detection started
- `GAMTrace-Parm2:<CustomURLsList>`
	* Meaning: Semicolon-separated custom SLO URLs configured
- `GAMTrace-Parm3:<SesURL>`
	* Meaning: Session URL used for matching
- `GAMTrace-&AppCliSLOURL:<SelectedURL>`
	* Meaning: Result: which custom URL was selected

---

## COMMON FAILURE POINTS
- Invalid URL After SLO (error 621): The `redirect_uri` sent by the client is not in the IDP Application's `ClientSingleLogoutValidURLsAfterSLO` list. The IDP refuses to redirect back
	* Fix: Add the client's post-logout URL to the Application's valid SLO URLs in Backoffice
	* The match is LITERAL — a trailing slash is significant. Read `GAMValidURLFromAList - Start_Method` to compare `Parm2` (configured list) against `Parm3` (URL being validated) before suspecting stale cache

- Behavior Configured to ClientOnly: User expects global logout but remains logged in at the IDP
	* Fix: Change `GAMRemoteLogoutBehavior` to `All` in Backoffice → Repository settings

- Missing State Parameter: Client throws "state not found" error
	* Fix: Check database connectivity. The state storage requires DB read/write access. Also check Redis if used

- URL-encoded FromURL (Java generators): On Java generators, the return URL may be stored URL-encoded (`http%3A%2F%2F…`). GAM then treats it as a relative path, producing a malformed redirect
	* Fix: See `domain-cross-generator.md` for Java-specific workarounds

- Custom SLO URL mismatch for non-GAM clients: The custom SLO URL list contains URLs for a different deployment than the one running (e.g., .NET Framework URL but .NET Core is active). The URL detection falls back to domain-level matching and selects a wrong endpoint
	* Fix: Ensure `ClientSingleLogoutCustomURLsSLO` in Backoffice contains the URL for the ACTIVE deployment

- Non-GAM SLO handler blocks the chain: The handler shows a page and waits for user interaction instead of auto-redirecting. The SLO chain stops
	* Fix: The handler MUST redirect back to `redirect_uri?state=<state>` immediately (no user click). See "Non-GAM Client SLO" below

---

## Non-GAM Client SLO
### Architecture
A non-GAM client is a GeneXus KB (or any app) that authenticates against GAM via HTTPClient/REST calls instead of using the GAM module. These clients:
- Call `oauth/gam/access_token` and `oauth/gam/userinfo` directly
- Store tokens in WebSession manually
- Have `ClientCallbackURLisCustom = true` in GAM's Application config
- Need custom SLO URLs because they don't have `/oauth/gam/signout`

### Custom SLO URL Selection
When GAM needs to send a signout to a non-GAM client, it selects which custom URL to use based on the session's original URL (the callback URL used during login)

Selection rules:
- If only one custom URL is configured → use it directly
- If multiple custom URLs are configured → progressive matching from most specific to least specific:
	* First: match at full application-path level (e.g., `http://host/appname/`)
	* Then: match at domain level (e.g., `http://host/`)
	* First match wins (order in the list matters!)
- If no match → error logged

CRITICAL: Order dependency. When multiple custom URLs share the same domain, the FIRST one in the list that matches at domain level wins. Always place the most specific (correct) URL first

CRITICAL: Deploy mismatch. If the list has a URL for .NET Framework but the running deploy is .NET Core, the matching algorithm won't find an exact match and falls to domain level, potentially selecting the wrong URL

### Differences Between Custom and Standard SLO Redirects
- Custom (non-GAM)
	* `server_ip` parameter: NOT sent — client has no GAM
	* Encryption: Never encrypted
	* Parameters sent: `client_id`, `redirect_uri`, `state`, `token`
- Standard (GAM client)
	* `server_ip` parameter: `server_ip=0` sent
	* Encryption: Encrypted if `ClientEncryptionKey` is set
	* Parameters sent: `client_id`, `redirect_uri`, `state`, `token`, `server_ip`, `repository`

### What the Non-GAM SLO Handler MUST Implement
The custom SLO handler (e.g., a GeneXus Web Panel) must:

- Receive parameters: `client_id`, `redirect_uri`, `state`, `token` (via querystring)
- Validate the token against its stored session (compare received `token` with stored `access_token`)
- Kill the local session (clear WebSession keys)
- Redirect IMMEDIATELY to `redirect_uri?state=<state>` — this is the return path to the IDP

CRITICAL: The redirect MUST be automatic (in Event Start or via HTTP 302). If the handler shows a page and waits for user interaction, the SLO chain breaks because the browser is the transport between nodes

Implementation shape: a Web Panel with `Parm(in:&client_id, in:&redirect_uri, in:&token, in:&state)`. On `Event Start`: compare the received `&token` against the stored session token (e.g., `&WebSession.Get("GAMToken")` parsed via `Oauth20AccessTokenSDT.FromJson`), clear the WebSession keys on match, then unconditionally `Link(Format("%1?state=%2", &redirect_uri, &state))` — no page render, no wait for user interaction

### Common Pitfalls for Non-GAM SLO Handlers
- Handler requires user click to redirect
	* Symptom: SLO chain stops at the non-GAM client page
	* Fix: Make redirect automatic in Event Start
- URL format `?&state=` instead of `?state=`
	* Symptom: Extra `&` may confuse parsers
	* Fix: Use `Format(!"%1?state=%2", …)`
- Token comparison fails
	* Symptom: Handler reports "TOKEN is not from this App"
	* Fix: Verify that the `token` sent by GAM matches the stored `access_token`. Check if SSORestToken is enabled — when SSO REST is active the stored value is the SSO REST token; see [domain-sso-rest.md — Single Logout interaction](../authentication/sso-rest/domain-sso-rest.md)
- Custom SLO URL points to wrong deployment
	* Symptom: 404 or wrong app receives the signout
	* Fix: Update `ClientSingleLogoutCustomURLsSLO` in Backoffice
- Custom SLO URL list has placeholder entries
	* Symptom: Wrong URL selected by domain match
	* Fix: Clean the list; only include real, reachable SLO endpoints
- `redirect_uri` is URL-encoded
	* Symptom: Double-encoding breaks the return URL
	* Fix: GeneXus `Format()` does not encode; use the value as-is

### Configuring Custom SLO URLs (Backoffice)
In GAM Backoffice of the IDP/sub-IDP:
- Navigate to Applications → select the non-GAM app
- Field: ClientSingleLogoutCustomURLsSLO (may appear as "Custom Single Logout URLs")
- Enter semicolon-separated URLs, one per deployment:
	```
	http://app-d.com/GAM_Dev_ClientDNetCoreSQL/slo;http://app-d.com/GAM_Dev_ClientDNetSQL/slo.aspx?isdebug=false
	```
- Ensure the URL for the currently active deployment is in the list
- Place more specific URLs first (important if multiple URLs share a domain)

---

## DIAGNOSTIC GUIDE
### SLO Not Reaching IDP
Symptom: User logs out from client but remains logged in at the IDP

Evidence to check:
- Client traces: Is `GAMTrace-SLO Redirect URL:` present? If absent, the client is not building the redirect
- IDP traces: Is `GAMTrace-==+++GAMExternalAuthenticationGAMRemote ====== START ---` present? If absent, the redirect is not reaching the IDP (network/URL issue)
- Is `GAMTrace-SLOProcess - IDP - Finish parent token:` present? If absent, `GAMRemoteLogoutBehavior` may be `ClientOnly`

Fix: Check `GAMRemoteLogoutBehavior` in Backoffice → Repository. Set to `All` for full SLO

### SLO Chain Stops at a Daughter
Symptom: Some clients are logged out but others remain active

Evidence to check:
- Check the IDP traces for the sequential notification loop — each daughter should appear
- If a daughter's signout URL returns a page (not a redirect), the chain stops there
- Check for pending sessions: `Status=Finished AND EndDate empty` in the GAM database indicates daughters that were marked but never notified

Diagnostic SQL:
```sql
-- Find pending SLO notifications (daughters marked but not yet notified)
SELECT SesGUID, SesAppId, SesSts, SesEndDate, SesURL
FROM GAMSession
WHERE SesParGUID = '<parent_session_guid>'
AND SesSts = 'F'
AND SesEndDate IS NULL;
```

Fix: Ensure all daughter applications respond to the signout with an automatic redirect. Check that custom SLO URLs are correct and reachable

### External IDP SLO Not Working
Symptom: GAM sessions are terminated but the external IDP (Microsoft, Keycloak) session persists

Evidence to check:
- Is `GAMTrace-GAMExternalAuthenticationSignout` present? If absent, GAM is not attempting external signout
- Is `GAMTrace-Oauth20 Redirect Signout URL:` present and correct? The URL should point to the external IDP's logout endpoint
- On return: Is `GAMTrace-GAMExternalAuthenticationInputValidParam - Otherwise Parm_State: SLO…` present? If absent, the external IDP is not redirecting back

Fix: Verify the external IDP's logout endpoint is correctly configured in GAM. Ensure `post_logout_redirect_uri` is registered in the external IDP's allowed redirect URIs

### Custom SLO URL Wrong Selection
Symptom: Non-GAM client receives signout at wrong URL (404 or wrong application)

Evidence to check:
- Check `GAMTrace-Parm2:` — the custom URL list
- Check `GAMTrace-Parm3:` — the session URL used for matching
- Check `GAMTrace-&AppCliSLOURL:` — the selected URL. Is it correct?

Fix: Reorder the URLs in `ClientSingleLogoutCustomURLsSLO` (Backoffice → Application) so the correct deployment URL appears first. Remove stale/placeholder URLs

---

## End-to-End SLO Flow Example (with Non-GAM Client)
```
IDP (user clicks logout)
	→ GAM initiates SLO, self-redirects as sub-IDP (server_ip=2)
	→ GAM marks all daughter sessions for notification
	→ GAM finds Client A → redirects browser to Client A's signout

Client A (GAM client, receives signout, has its own children)
	→ GAM escalates from client to sub-IDP (server_ip: 0 → 2)
	→ GAM kills own session, marks its daughters for notification
	→ GAM finds Client D (non-GAM) → redirects to custom SLO URL

Client D (non-GAM, receives signout)
	→ Validates token, kills local session
	→ AUTO-REDIRECTS back to Client A (redirect_uri?state=…)

Client A (receives callback from Client D)
	→ GAM finds next daughter: Client B → redirects to Client B's signout

Client B (GAM client, receives signout)
	→ GAM kills local session → redirects back to Client A

Client A (all daughters processed)
	→ GAM redirects back to IDP with original state

IDP (receives return from Client A)
	→ SLO complete → redirects to post-logout page
```

---

## Event Subscriptions Are Synchronous and Blocking
`ExecuteEventSubscriptions` fires the `repository-logout` event synchronously during SLO. It BLOCKS the SLO response until every registered listener completes. There is NO default timeout

Risk:
- A slow listener delays the redirect to the next daughter (or the return to the IDP)
- A hung listener stops the SLO chain entirely — no fallback, no skip

Detect in trace (v18u15+):

```
genexus.security.api.ExecuteEventSubscriptions - Start_Sub-ExecuteExternalObject - {"data":{…}}
```

…with no matching `End_Method` (or no matching end of the external call) for an extended time

Handler contract and diagnostics for a listener that never runs: [GAM Events Subscription: Handler Contract](../events/domain-event-subscription.md)

Mitigation guidance:
- Keep `repository-logout` listeners idempotent and short
- Move long-running work out-of-band (queue, async job, separate process)
- Monitor listener duration as part of SLO health

---

## Under-Appreciated Gap: `gam.SessionToken` Rows Survive SLO
During SLO, GAM cleans:
- The WebSession entry (browser cookie state)
- The `gam.Session` row (`Status = 'F'`, `EndDate` set once the daughter is notified)

GAM does NOT delete rows in `gam.SessionToken`. If an application stores server-side tokens keyed off `SessionToken`, those rows persist after SLO completes. They are only removed by out-of-band cleanup (scheduled job, manual query, retention policy)

Impact:
- A stale `gam.SessionToken` row can be reused if the application does not re-validate against `gam.Session.Status`
- Token-lookup endpoints that join only on `gam.SessionToken` will still resolve
- Auditing/retention processes must explicitly target `gam.SessionToken`

Validate post-SLO: `gam.Session` shows `F` + `EndDate` while the matching `gam.SessionToken` rows are still present

---

## Under-Appreciated Gap: Cache Keys Survive SLO
SLO does NOT purge GAM's in-process caches. Keys that survive include:
- `com.genexus.gam.applications.repository_<RepositoryGUID>`
- `com.genexus.gam.eventsubs.repository_<RepositoryGUID>`
- Auth-type caches (per provider configuration)

Risk: if an administrator changes an external IDP provider's logout endpoint (or event subscription, or application setting) DURING an SLO flow, GAM uses the cached old value. The SLO redirect may be built with the previous endpoint even though the DB already has the new one

Mitigation:
- Treat provider-endpoint changes as deploy-time changes; restart workers or force a cache refresh after changes
- When diagnosing "config change didn't take effect", confirm cache state before blaming the DB
- Document cache TTLs alongside provider configuration so operators know when the new value becomes effective

---

## Return-Trip Handler: `/oauth/gam/callback?state=SLOExt<hash>`
When the external IDP finishes its signout, it redirects the browser back to GAM at `/oauth/gam/callback`. The query string contains ONLY `state` — no `code`, no `OriginType`. This is EXPECTED — external IDPs do not know GAM-internal fields

Observable sequence at the callback handler:
- `GAMExternalAuthenticationInputLoadParam` reads `Parm_OriginType:""` (empty)
- `GAMExternalAuthenticationInputValidParam` falls to the `Otherwise` branch
- GAM queries `LoginTmpTokenState` using the hash of the incoming `state`
- GAM loads the snapshot, hydrates the SDT with `isIdentityProvider:true, isReturnFromClient:true`
- Control passes to `RepositoryInputExternalAuthentication`, which enters the `ValidWhenIsIDP-True` sub-branch
- GAM builds the final redirect: `<FromURL>?state=SLOExt<hash>`

Failure to reach step 3 (no snapshot row found) always produces a silent fallback to the login screen. See the silent-failure catalog in `domain-slo-external-idp.md`

---

## Related Files
- [GAM-as-SP — OAuth 2.0 Common Flow (observable in trace)](../authentication/external-providers/oauth20/common/oauth20-common-flow.md) — shared `state` / callback machinery and trace format used by both login and SLO
- [SLO External IDP Analysis Playbook](domain-slo-external-idp.md) — external IDP playbook and silent-failure catalog
- [Debugging SLO with Non-GAM Clients](slo-non-gam-debugging.md) — diagnosing non-GAM clients in the SLO chain

---

## DECISION POINTS
Before configuring or diagnosing SLO, check which Decision Points apply

### DP-1: SLO Scope
- Trigger: User asks about logout or SLO configuration
- Question: "What type of SLO do you need? `local` (close only the current session), `slo_gam` (close all GAM client sessions), or `slo_full` (include non-GAM clients and external IDPs)?"
- Options:
	* `local` — Close only the current app session. Simple, no chaining
	* `slo_gam` — (DEFAULT) Close sessions in all registered GAM clients
	* `slo_full` — Include non-GAM clients (custom SLO URL) and external IDPs. Requires additional Backoffice configuration
- Impact:
	* `local`: No SLO chain
	* `slo_gam`: Requires clients registered as GAM applications with SLO enabled
	* `slo_full`: Requires `ClientSingleLogoutCustomURLsSLO` for non-GAM clients + external IDP logout endpoint configured
- Phase: configure

### DP-2: Non-GAM Clients?
- Trigger: When DP-1 = slo_full, or user mentions "non-GAM", "HTTPClient"
- Question: "Do you have non-GAM clients (apps without the GAM module) that need to participate in SLO?"
- Options:
	* `no` — (DEFAULT) All clients have GAM. Standard SLO
	* `yes` — Non-GAM clients exist. Require: custom SLO URL in Backoffice, logout handler in the client, auto-redirect back to IDP
- Impact: Non-GAM clients require a custom handler that receives the signout and redirects back automatically. Without auto-redirect, the SLO chain breaks
- Phase: configure

### DP-3: GAMRemoteLogoutBehavior
- Trigger: When configuring SLO at repository level
- Question: "What remote logout behavior? `All` (default, notifies all clients), `ClientAndIP` (only client and IDP), or `ClientOnly` (only the requesting client)?"
- Options:
	* `All` — (DEFAULT) Notifies ALL clients (GAM and non-GAM). Most secure
	* `ClientAndIP` — Closes session at client and IDP, but does not notify other clients
	* `ClientOnly` — Only closes the requesting client's session. IDP session persists
- Impact: Repository property `GAMRemoteLogoutBehavior`. Affects which daughter sessions are terminated during SLO
- Phase: configure
