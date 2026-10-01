---
name: domain-slo-external-idp
description: Playbook for analyzing SLO flows with external IDPs
---

# SLO External IDP Analysis Playbook
This is the LOGOUT direction of an IdP chain. For the LOGIN direction (`auth:` Local Login URL proxy), see [IDP→IDP Login Chaining — `auth:` Local Login URL Proxy](../authentication/local-idp/idp-chaining.md)

## When to Use
Use this playbook when debugging an SLO (Single Log Out) flow where the GAM IDP redirects to an external IDP (Microsoft, Keycloak, Google, etc) as part of the logout process

---

## Architecture: SLO with External IDP (Chained / IDP-of-IDP)
When the GAM IDP itself authenticates users via an External IDP (e.g., Microsoft OAuth 2.0, Keycloak, Google), the SLO flow adds an intermediate redirect cycle

### State Snapshot Mechanism
GAM uses the `state` parameter as a context bus between redirects. When an external IDP redirect is needed:
- GAM takes a snapshot of the accumulated context (tokens, return URL, LogoutType, etc.)
- Serializes it as JSON and stores it in the database, keyed by a hash of the state string
- Sends only the state key to the external IDP via `post_logout_redirect_uri`
- When the external IDP returns, GAM retrieves the snapshot from the database and restores context

### State Persistence — `LoginTmpTokenState` DB Table
The snapshot JSON is written to the `LoginTmpTokenState` table, keyed by a hash of the `state` string, with a configured TTL. Observable columns include the state hash key, the serialized payload, and an expiration timestamp

Required payload fields inside the JSON:
- `GAMTokenState` — current token identifier for the SLO leg
- `RepositoryGUID` — repository scope
- `ApplicationId` — originating application
- `FromURL` — final destination (stored DECODED, see Java generator caveat)
- `LogoutType` — `3` for signout flows
- `EndLogout` — `1` on the final leg

TTL failure mode: if the TTL expires before the return trip completes, the callback lands in the `Otherwise` branch of `GAMExternalAuthenticationInputValidParam` with no snapshot available. The user is silently redirected to the login screen instead of `FromURL`. No error is raised to the user

Diagnostic: query `LoginTmpTokenState` for the specific state hash. If the row is absent, either the write failed at snapshot creation or the TTL already expired

### State Snapshot JSON Structure (SLO)
```json
{
  "GAMTokenState": "SLOInt…",
  "RepositoryGUID": "bbe58eb2-…",
  "ApplicationId": 5,
  "AuthenticationTypeName": "oauth20-microsoft",
  "FromURL": "http://example.com/Client/afterlogoutobject.aspx",
  "SessionToken": "bbe58eb2-…!abc…",
  "ParentToken": "bbe58eb2-…!xyz…",
  "LogoutType": 3,
  "EndLogout": 1
}
```

Key fields:
- `FromURL`: The final destination URL after SLO completes. Must be stored decoded (not URL-encoded)
- `LogoutType: 3`: Indicates this is a signout flow
- `EndLogout: 1`: Indicates this is the final leg of the logout (return from external IDP)

### 4-Step Flow: Client → IDP GAM → External IDP → IDP GAM → Client
#### Step 1: Client initiates logout
- Client calls `/oauth/gam/signout?client_id=…&server_ip=1&state=SLO…&token=…&redirect_uri=…`
- Standard GAMRemote SLO initiation

#### Step 2: IDP GAM receives signout, kills sessions, redirects to external IDP
- GAM identifies the request as an SLO signout
- CRITICAL: GAM kills its own sessions AND children's sessions HERE, BEFORE redirecting to the external IDP
- GAM stores the state snapshot in the database with `LogoutType:3`, `EndLogout:1`
- GAM builds a redirect URL to the external IDP's logout endpoint:
	```
	https://external-idp.com/logout?state=SLO…&post_logout_redirect_uri=https://gam-idp.com/oauth/gam/callback
	```
- The `post_logout_redirect_uri` points to GAM's own callback endpoint, NOT the client's

#### Step 3: External IDP returns to IDP GAM callback
- External IDP redirects to `/oauth/gam/callback?state=SLO…`
- GAM receives only the `state` parameter — this is expected (no OriginType)
- GAM reads the state key, retrieves the snapshot from the database, restores full context
- Does NOT need to kill sessions — they were already killed in Step 2
- GAM reads `FromURL` from the snapshot and produces the final redirect

#### Step 4: Client receives the final redirect
- Client receives redirect to `FromURL?state=SLO…`
- Logout is complete

---

## Sub-IDP Escalation Pattern (Client with Daughter Sessions)
When a GAM client receives an SLO signout (client role) but itself has daughter sessions (it acts as IDP for other clients), it escalates from client role to sub-IDP role within the same request

### How Escalation Works
- GAM receives the signout as a client (`server_ip=0`)
- GAM looks up its own session and finds it is a root session with active children
- GAM saves both the original IDP context and its own context to the database
- GAM changes its role from client (0) to sub-IDP (2)
- GAM kills its own session
- GAM then processes as an IDP: marks all daughter sessions for notification, and begins the sequential redirect chain to each daughter

### Clarification: `server_ip` 0 → 2 Is In-Request, Not a Round-Trip
The `server_ip` escalation from `0` to `2` happens within the SAME HTTP request to `/oauth/gam/signout`. It is NOT a second redirect or a second request. During processing, GAM detects active daughter sessions in memory and in the DB and switches its control flow to the daughter-notification branch before producing the outbound redirect

State machine summary:
- Level `0` — client (own session only, no daughters)
- Level `1` — IDP (kills parent + broadcasts to daughters)
- Level `2` — sub-IDP (started as client, escalated mid-request after detecting daughters)

Reading traces: you will see two `SLOProcess` markers within a single request boundary — the first with `Parm_isServerIP:0   Parm_FirstCall:0`, the second with `Parm_isServerIP:2   Parm_FirstCall:1`. Do not treat these as separate hops

### State Entries Created During Escalation
The sub-IDP creates TWO state entries in the database:

- IDP state
	* Purpose: Stores the return path to the original IDP (original state, IDP signout URL)
- Client state
	* Purpose: Stores context for daughter processing (return URL, session token, LogoutType, cross-reference to IDP state)

### How the Chain Returns After All Daughters
When all daughters have been processed and returned:
- GAM detects no more pending daughters
- GAM redirects to the return URL stored in the client state, including the cross-reference to the original IDP state
- The original IDP receives this as a return from the client, completes the SLO, and redirects to the final post-logout page

---

## Daughter Session Notification (Two-Phase)
GAM uses a two-phase approach for daughter session cleanup:

### Phase 1: Bulk Mark
GAM marks ALL active daughter sessions as Finished in the database, but intentionally does NOT set the EndDate timestamp. Sessions with status=Finished and no EndDate are "pending SLO notification."

### Phase 2: Sequential Notify
GAM finds the NEXT daughter that needs notification (Finished status, empty EndDate), sets the EndDate to mark it as "being processed," and redirects the browser to that daughter's signout URL. When the daughter returns, GAM picks the next pending daughter. This repeats until no more pending daughters remain

### Key Behavioral Rules
- `Status=Finished AND EndDate empty` = pending SLO notification (session killed in DB, client not yet notified)
- `Status=Finished AND EndDate set` = already processed (session killed AND client was notified)
- The loop iterates by browser redirect callbacks: redirect to daughter → daughter returns → next iteration
- If a daughter never returns (broken URL, non-responding client), the chain stops. There is NO timeout or skip mechanism
- GAM skips the application that INITIATED the signout (to avoid circular redirect) and any application with SLO disabled

## Orchestration Entry: `Logout_internal`
Each GAM node in the chain is driven by `Logout_internal`. Signature:

```
Logout_internal(CacheRepository, isForceLogout, SessionType, FinishSession, GAMState, &RedirURL, &isOK, &Errors)
```

It calls, in order: `GAMRemoteLogout` (kills IDP sessions), `GAMExternalAuthenticationSignout` (builds the external IDP redirect), `GAMStateClientAPI` (persists the snapshot to `LoginTmpTokenState`), and returns `&RedirURL` to the HTTP handler. Use its `Start_Method` / `End_Method` markers to bracket each hop when reading a trace

## Critical: `post_logout_redirect_uri` Points to GAM, Not the Client App
When GAM-as-IDP sends the browser to the external IDP's logout endpoint, `post_logout_redirect_uri` MUST point to GAM's own callback: `/oauth/gam/callback`. It MUST NOT point to the client application's post-logout URL

Why:
- The external IDP redirects back to GAM first
- GAM restores SLO chain state from `LoginTmpTokenState`
- GAM then redirects to the final `FromURL`

If clients or administrators misconfigure `post_logout_redirect_uri` to the client app URL directly, the SLO state machinery cannot resume. The state snapshot is never re-read, downstream daughters are never notified, and `FromURL` is never reached cleanly

Configuration check: wherever `post_logout_redirect_uri` is built or overridden (application-level override, provider override), confirm the value resolves to `<GAM-IDP-host>/oauth/gam/callback`

## Prerequisites
You need 4 logs, one for each step of the flow. Ask the user to provide traces for:

- Client to IDP GAM: The client GAMRemote initiating logout (`LogoutInternal`)
- IDP GAM receives signout: The IDP GAM processing the `/oauth/gam/signout` request
- External IDP to IDP GAM callback: The IDP GAM receiving the callback from the external IDP
- IDP GAM to Client: The client receiving the final redirect

---

## Analysis Steps
### Step 1: Verify Client Initiation
Search for:
```
GAMTrace-Logout_internal -==LogoutGAMRemote
GAMTrace-Logout_internal &RedirToURL:
```

Validate:
- [ ] `LogoutType: 1` in the `SessionPar`
- [ ] The `&RedirToURL` points to the IDP's `/oauth/gam/signout` endpoint
- [ ] `redirect_uri` parameter is URL-encoded correctly
- [ ] `server_ip=1` is present (confirming the target is IDP)

### Step 2: Verify IDP GAM Processing
Search for:
```
GAMTrace-GAMExternalAuthenticationInputLoadParam: &OriginType: oauth
GAMTrace-GAMExternalAuthenticationInputLoadParam: &OriginTypeValue: signout
GAMTrace-GAMExternalAuthenticationSignout
GAMTrace-Oauth20 Redirect Signout URL:
```

Validate:
- [ ] `OriginType: oauth` and `OriginTypeValue: signout` are correctly detected
- [ ] `GAMRemoteLogout` is executed (sessions killed)
- [ ] `GAMExternalAuthenticationSignout` is called with correct Signout config
- [ ] `post_logout_redirect_uri` points to GAM's OWN callback (`/oauth/gam/callback`), NOT the client's
- [ ] The state key starts with `SLO`

### Step 3: Verify External IDP Callback (CRITICAL)
Search for:
```
GAMTrace-GAMExternalAuthenticationInputLoadParam: &OriginType:
GAMTrace-GAMExternalAuthenticationInputValidParam - Otherwise Parm_State: SLO
GAMTrace-GAMExternalAuthenticationInputValidParam-&APIStateClientPar:
GAMTrace-GAMExternalAuthenticationInput &RedirToURL OK:
```

Validate:
- [ ] `OriginType` and `OriginTypeValue` are empty (this is EXPECTED)
- [ ] `ValidParam` falls into `Otherwise` and loads state from DB
- [ ] `APIStateClientPar` contains `LogoutType: 3` and `EndLogout: 1`
- [ ] CRITICAL: `FromURL` in `APIStateClientPar` is NOT URL-encoded (should be `http://…` not `http%3A%2F%2F…`)
- [ ] `isIdentityProvider: true` and `isReturnFromClient: true` in the SDT
- [ ] Enters `ValidWhenIsIDP-True`
- [ ] `&RedirToURL OK:` shows the correct absolute URL to the client

### Step 4: Verify Client Receives Redirect
Search for:
```
AbsoluteUri dynamicport:
```

Validate:
- [ ] The URL is the correct `afterlogoutobject` (or equivalent) with `state=SLO…`
- [ ] No URL concatenation artifacts

---

## Common Issues and Solutions
- Symptom: `OriginType` empty on callback
	* Cause: Normal — external IDP only returns `state`
	* Solution: No action needed
- Symptom: `FromURL` is URL-encoded in `APIStateClientPar`
	* Cause: Java generator encoding bug
	* Solution: Report to GAM team — no client workaround
- Symptom: Concatenated redirect URL
	* Cause: `Link()` treats encoded URL as relative
	* Solution: Same as above
- Symptom: `SLOProcess` traces missing on callback
	* Cause: Sessions already killed in Step 2
	* Solution: Normal behavior — not a bug
- Symptom: `post_logout_redirect_uri` points to client URL
	* Cause: IDP should redirect to its own callback
	* Solution: Bug in GAM signout URL construction

---

## Silent-Failure Catalog (External IDP SLO)
Each entry pairs a condition with its trace signature, symptom, root cause, and fix. These are the most common "nothing-obvious-broke-but-SLO-didn't-complete" scenarios

- Condition: `state` missing on return from external IDP
	* Trace signature: callback hits `/oauth/gam/callback` with no `state` query parameter; `GAMExternalAuthenticationInputLoadParam` logs empty `Parm_State`
	* Symptom: user stuck at the external IDP post-logout page or redirected to the wrong URL
	* Root cause: external IDP stripped `state`, or the signout URL was built without `Signout.State_Include = True`
	* Fix: set `Signout.State_Include = True` in the auth-type Signout config so GAM always includes `state` in the outbound URL

- Condition: `post_logout_redirect_uri` not whitelisted at the external IDP
	* Trace signature: `GAMTrace-Oauth20 Redirect Signout URL:` shows the outbound URL built, but no callback reaches `/oauth/gam/callback`
	* Symptom: external IDP displays "redirect_uri not approved" (or equivalent); no callback traces on the GAM IDP
	* Root cause: the external IDP's allowed-post-logout list does not include GAM's callback
	* Fix: register `<GAM-IDP-host>/oauth/gam/callback` in the external IDP's allowed post-logout redirect URIs

- Condition: state snapshot not found in `LoginTmpTokenState`
	* Trace signature: callback reaches `Otherwise Parm_State: SLO…` but `APIStateClientPar` is empty or errors out
	* Symptom: user silently redirected to the login screen instead of `FromURL`
	* Root cause: TTL expired before the return trip completed, OR the snapshot INSERT never happened (DB write failure upstream)
	* Fix: check DB logs for failed inserts around the time of logout; confirm the TTL for `LoginTmpTokenState` is larger than the expected round-trip duration of the external IDP's logout flow

- Condition: `FromURL` stored URL-encoded (historical Java-generator bug)
	* Trace signature: `APIStateClientPar` shows `FromURL` as `http%3A%2F%2Fhost/…` instead of `http://host/…`
	* Symptom: final redirect malformed; browser resolves it as a relative path and concatenates to the current URL
	* Root cause: Java generator encoded the URL before persisting the snapshot
	* Fix: upgrade the generator, or apply a KB-level workaround that decodes `FromURL` before storing it to `LoginTmpTokenState`

- Condition: `end_session_endpoint` / `Signout.URL` not configured
	* Trace signature: `GAMTrace-Oauth20 Redirect Signout URL:` shows an empty or base-only URL with no logout path
	* Symptom: redirect to external IDP fails; browser lands on an unrelated page
	* Root cause: the provider's logout endpoint is missing from the auth-type config
	* Fix: configure `Signout.URL` (or OIDC `end_session_endpoint`) in the auth-type Backoffice page

- Condition: `id_token_hint` missing (OIDC providers)
	* Trace signature: external IDP accepts the signout request but logs "token not validated" or does not terminate the session
	* Symptom: orphan session at the external IDP; subsequent logins bypass re-authentication
	* Root cause: OIDC providers require `id_token_hint` to validate the logout request
	* Fix: ensure `id_token` is stored at login time and passed to signout; confirm the provider's config requests `id_token` in the initial login response

- Condition: non-GAM custom SLO URL misordered
	* Trace signature: `GAMRemotLogoutDetectAppSLOURL` picks a domain-level match that is not the active deployment
	* Symptom: signout reaches the wrong endpoint (404, or wrong deployment receives the request)
	* Root cause: the `ClientSingleLogoutCustomURLsSLO` list has multiple entries sharing a domain and the wrong one is first
	* Fix: reorder `ClientSingleLogoutCustomURLsSLO` so the correct deployment URL appears first in the list

- Condition: non-GAM handler does not auto-redirect
	* Trace signature: no callback return trace after the signout was sent to the non-GAM client
	* Symptom: SLO chain stops at the non-GAM handler; subsequent daughters never notified
	* Root cause: the handler renders a page and waits for user interaction instead of emitting HTTP 302
	* Fix: fix the handler to emit an immediate 302 to `redirect_uri?state=<state>`; never show a page in the auto path

- Condition: `GAMRemoteLogoutBehavior = ClientOnly`
	* Trace signature: `Finish parent token:` absent on the IDP; no daughter notification loop
	* Symptom: user logs out of one client but remains logged in at the IDP and on every other daughter
	* Root cause: repository configured to terminate only the requesting client
	* Fix: change `GAMRemoteLogoutBehavior` to `All` in Backoffice → Repository

- Condition: expired external token at signout time
	* Trace signature: signout proceeds but the external IDP response reports "token invalid"
	* Symptom: local sessions cleaned but external IDP session persists
	* Root cause: the stored external token has expired before the logout attempt
	* Fix: refresh the token before calling signout when possible, or skip the external signout when `GAMValidExternalTokenExpires` returns true (session is already effectively invalidated externally)

---

## Related Files
- [GAM-as-SP — OAuth 2.0 Common Flow (observable in trace)](../authentication/external-providers/oauth20/common/oauth20-common-flow.md) — shared `state` / callback machinery and trace format
- [GAM Single Log Out (SLO)](domain-slo.md) — SLO base flow, session hierarchy, `Logout_internal`, `LoginTmpTokenState`
- [Debugging SLO with Non-GAM Clients](slo-non-gam-debugging.md) — non-GAM client diagnostics
