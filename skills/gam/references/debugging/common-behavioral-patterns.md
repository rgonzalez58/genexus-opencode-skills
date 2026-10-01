---
name: common-behavioral-patterns
description: Verified behavioral patterns observable in traces — state prefixes, decision trees, flow phases
---

# GAM Behavioral Patterns
## DEFINITION
Observable behavioral patterns used for diagnostics. All patterns described here are verified through trace analysis and represent behavior the user can observe in GAM log output

---

## State Prefixes (`GAMTokenState`)
The `state` parameter prefix determines the flow type:

- `GRESTD`
	* Flow Type: Login / REST Authentication
	* Example: `GRESTD<random>`
- `SLOInt`
	* Flow Type: Single Log Out (Internal/Chained)
	* Example: `SLOInt<random>`
- `OA2STD`
	* Flow Type: OAuth 2.0 Standard (IDP callback from Microsoft)
	* Example: `OA2STD<random>`

### How to use
When you see a `state` parameter in a log, the prefix tells you immediately what flow you are in:
- `state=GRESTD…` — This is a login/auth flow
- `state=SLOInt…` — This is an SLO flow
- `state=OA2STD…` — This is a return from OAuth 2.0 external IDP

---

## Request Routing (`OriginType` + `OriginTypeValue`)
These two fields determine the entire routing of the authentication input handler:

- `OriginType`=`oauth`, `OriginTypeValue`=`auth`, `isIDP`=`true`
	* Flow: IDP authentication path → remote logout handler
	* Description: Client requesting authentication at IDP
- `OriginType`=`oauth`, `OriginTypeValue`=`signout`, `isIDP`=`true`
	* Flow: IDP authentication path → remote logout → SLO processing
	* Description: Client requesting SLO at IDP
- `OriginType`=`oauth`, `OriginTypeValue`=`auth`, `isIDP`=`false`
	* Flow: Non-IDP path → external IDP login flow
	* Description: Return from IDP with auth code
- `OriginType`=(empty), `OriginTypeValue`=(empty), `isIDP`=derived
	* Flow: State recovery from DB → derive context from stored snapshot
	* Description: Return from external IDP (callback)

### Key: When `OriginType` is empty
This happens when the external IDP (Microsoft, Keycloak) returns to GAM's callback URL. The external IDP only sends back `state=SLO…` — no `oauth=` parameter. GAM detects this, falls into the state recovery path, reads the state from DB, and derives `OriginType` from the stored snapshot

---

## Logout Flow Phases (`GAMLogoutType` Progression)
The `GAMLogoutType` value evolves as the SLO flow progresses:

```
Phase 1 — Client initiates logout
	→ GAMLogoutType = 1 (client-only logout, redirect to IDP pending)

Phase 2 — IDP receives signout request, inspects auth type
	→ GAMLogoutType = 2 (signout received, checking if external IDP SLO needed)

Phase 3 — Auth type has SLO enabled for external IDP
	→ GAMLogoutType = 3 (external IDP SLO required, store snapshot, redirect to external IDP)

If the auth type does NOT have SLO enabled:
	→ GAMLogoutType stays at 2 (no external redirect needed)
```

### Trace signatures for each transition
```
GAMTrace- Signout AuthTypeName:gamremote   &GAMLogoutType:1
GAMTrace-GAMExternalAuthenticationInputValidParam - &GAMLogoutType:2   &SesAutTypeName: oauth20
GAMTrace- Signout AuthTypeName:oauth20   &GAMLogoutType:3
```

### Summary
- Value `1`: Client-only logout. Next action: Redirect to IDP `/oauth/gam/signout`
- Value `2`: IDP received signout, checking auth type. Next action: Check if auth type has `SLOEnable`
- Value `3`: External IDP SLO required. Next action: Store snapshot, redirect to ext. IDP, then return

---

## SLO Processing Flow (IDP Side)
When the IDP executes SLO processing with `Parm_isServerIP: 1`, it follows this sequence:

```
1. Find the child session token (TokenToFinish)
2. Find the parent IDP session token (ParentToken)
3. Finish the child token
4. Check GAMRemoteLogoutBehavior:
	 - If "cliip" or "clial" → also finish the parent token
	 - If "clionl" → skip parent token
5. End all daughter sessions (bulk mark)
6. Redirect to initial SLO origin
7. Check LogoutType:
	 - If 3 → load IDP session from DB, check auth type SLO settings, build external IDP logout URL, redirect to external IDP
	 - If not 3 → direct redirect to FromURL
```

### Trace signatures for each step
```
GAMTrace-SLOProcess - Parm_isServerIP:   1   Parm_FirstCall:   1
GAMTrace-SLOProcess - IDP - &TokenToFinish:<TOKEN>
GAMTrace-SLOProcess - IDP - SLO process initiated in Client - &ParentToken:<TOKEN>
GAMTrace-SLOProcess - IDP - Local token to finish:<TOKEN>
GAMTrace-SLOProcess - IDP - Finish token:<TOKEN>
GAMTrace-SLOProcess - IDP - Finish parent token:<TOKEN>
GAMTrace-SLOProcess -  ==IDP-EndDaughtersSessions ++++++
GAMTrace-SLOProcess - ==IDP-RedirectToInitialSLO ++++++
GAMTrace-SLOProcess - LogoutType:   3   &APIStateClientPar.FromURL:<URL>   GAMRemoteLogoutBehavior:<BEHAVIOR>
GAMTrace-SLOProcess - ==LoadSessionFromDB ++++++
GAMTrace-SLOProcess - &Token:<PARENT_TOKEN>
```

---

## `Logout_internal` Stages
There are TWO calls to `Logout_internal` in a chained SLO:

### `Logout_internal-1` (Client side)
Triggered by the client GAM application's logout. Kills local session, builds redirect to IDP
```
GAMTrace-Logout_internal-1 &SessionPar:{"Mode":"FINISH",...,"LogoutType":1,...}
GAMTrace-Logout_internal -==LogoutGAMRemote ++++++
GAMTrace-Logout_internal &ServerURL:<IDP_URL>
GAMTrace-Logout_internal &RedirAfterLogoutURL:<CLIENT_AFTERLOGOUT_URL>
GAMTrace-Logout_internal &GAMRemoteServer:<IDP_SIGNOUT_URL>
GAMTrace-&RedirToURL:<IDP_SIGNOUT_URL_WITH_PARAMS>
```

### `Logout_internal-2` (IDP side, during SLO processing)
Triggered by the IDP after killing sessions, when `LogoutType=3` and the auth type has `SLOEnable=true`
```
GAMTrace-Logout_internal-2 &SessionPar:{"ApplicationId":2,"ExternalToken":"<JWT>",...}
GAMTrace-Logout_internal The autentication type: oauth20 has Single Logout Enable
```

---

## Signout State Generation
When the IDP receives a signout request, it determines if it needs to redirect to an external IDP:

```
1. Read the session associated with the token being finished
2. Identify the authentication type used for the original login
3. Set GAMLogoutType = 2 (initial value)
4. Check: does the authentication type have SLO enabled?
	 - YES → Escalate GAMLogoutType to 3, build state snapshot with LogoutType:3, store in DB
	 - NO  → Keep GAMLogoutType at 2, build state snapshot with LogoutType:2
5. Continue to SLO processing
```

### Trace signatures
```
GAMTrace-GAMExternalAuthenticationInputValidParam ==GenerateSignoutState ++++++
GAMTrace-GAMExternalAuthenticationInputValidParam - &GAMLogoutType:2   &SesAutTypeName: oauth20   &ServerToken:<TOKEN>
GAMTrace- Signout SLOEnable:true   ExternalToken:<JWT>
GAMTrace- Signout AuthTypeName:oauth20   &GAMLogoutType:3
```

---

## External IDP Callback Flow
When GAM receives a callback from an external IDP (only `state=SLO…` in querystring):

```
1. Parameters arrive empty: OriginType = (empty), OriginTypeValue = (empty)
2. Routing falls into the "Otherwise" path
3. State prefix "SLO" is detected
4. State snapshot is loaded from the temporary storage table
5. Context is derived: isIdentityProvider = true, isReturnFromClient = true
6. State is deleted from DB (cleanup)
7. Enters IDP processing path
8. Reads FromURL from the stored state snapshot
9. Builds redirect response back to the original SLO flow
```

### Trace signatures
```
GAMTrace-GAMExternalAuthenticationInputLoadParam: &OriginType:
GAMTrace-GAMExternalAuthenticationInputLoadParam: &OriginTypeValue:
GAMTrace-GAMExternalAuthenticationInputValidParam - Otherwise Parm_State: SLO...
GAMTrace-GAMExternalAuthenticationInputValidParam ==ValidStateInDB ++++++ SLO...
GAMTrace-GAMStateClientAPI - Delete State:SLO...
GAMTrace-GAMExternalAuthenticationInputValidParam - isIdentityProvider: true   isReturnFromClient:true
GAMTrace-GAMExternalAuthenticationInput ==ValidWhenIsIDP-True ++++++
GAMTrace-GAMExternalAuthenticationInput &RedirToURL OK:<FromURL>?state=SLO...
```

---

## `GAMRemoteLogoutBehavior` Values
- `clionl` (Client Initiated Online)
	* Kills Client Session: Yes
	* Kills IDP Session: No
	* Kills Daughter Sessions: No
- `cliip` (Client and IP)
	* Kills Client Session: Yes
	* Kills IDP Session: Yes
	* Kills Daughter Sessions: No
- `clial` (All)
	* Kills Client Session: Yes
	* Kills IDP Session: Yes
	* Kills Daughter Sessions: Yes

### Where to find it in logs
```
GAMTrace-SLOProcess - LogoutType: 3   &APIStateClientPar.FromURL:<URL>   GAMRemoteLogoutBehavior:<VALUE>
```

---

## State Storage Structures (`APIStateClientPar` vs `APIStateIDPPar`)
GAM uses TWO different state storage structures:

- `APIStateClientPar`
	* Stored by: Client GAM or IDP during SLO
	* Used for: Preserving client context across IDP redirects
	* Key fields: `FromURL`, `LogoutType`, `EndLogout`, `AuthenticationTypeName`
- `APIStateIDPPar`
	* Stored by: IDP GAM
	* Used for: Preserving IDP context during authentication
	* Key fields: `ClientURL`, `LocalLoginURL`, `ScopesList`, `IDPType`, `ApplicationId`

### `APIStateIDPPar` structure (Login)
```json
{
  "State": "GRESTD...",
  "Type": "auth",
  "ApplicationId": 4,
  "LocalLoginURL": "http://.../gamexampleidplogin",
  "ClientId": "...",
  "ClientURL": "http://.../oauth/gam/callback",
  "ScopesList": "gam_user_data+gam_user_roles",
  "IDPType": 1
}
```

### `APIStateIDPPar` structure (Signout)
```json
{
  "State": "SLOInt...",
  "Type": "signout",
  "ApplicationId": 4,
  "LocalLoginURL": "http://.../gamexampleidplogin",
  "ClientId": "...",
  "ClientURL": "http://.../afterlogoutobject"
}
```

---

## Shared-DB State Collision — `IDP-` Prefix
Observable diagnostic pattern for the u13HF/u14 shared-DB regression. For the full mechanism and version matrix, see [GAM Session and Token Reference](../authentication/external-providers/common/domain-session-token.md) — section "CRITICAL: IDP- Prefix Mechanism"

### How to diagnose in logs
```
ERROR GeneXus.Data.ADO.GxCommand - Return GxCommand.ExecuteNonQuery Error
	Infraccion de la restriccion PRIMARY KEY 'PK__LoginTmp__...'
	...gam.LoginTmp...
	ObjectName:GeneXus.Security.API.gamstateidpapi__default
```

Key indicators:
- `gamstateidpapi` in the stack trace (confirms the IDP-side handler is inserting)
- The duplicate key value corresponds to a `GRESTD` or `SLOInt` state hash
- `error_code=532` in the redirect back to the client

---

## Authorization Header Silent Skip
Verified Pattern: When GAM builds outgoing requests to external IDPs, it checks whether each response field is non-empty before adding the corresponding HTTP header. If a field is empty, the header is silently omitted — no trace line is emitted

### The Authorization Header Behavior
```
Token Response received → GAM reads token_type
	- token_type NOT empty → Authorization header is added, trace line emitted
	- token_type IS empty  → entire header block skipped silently
													→ no trace line emitted
													→ userinfo request sent WITHOUT Authorization header
													→ IDP returns Code 114: "Token no valido"
```

### Trace Signatures — What to Look For
```
Normal (token_type = "Bearer"):
GAMTrace-GAMRemote Token Response:{"access_token":"xxx","token_type":"Bearer",...}
GAMTrace-GAMRemote AddHeader: Authorization:Bearer xxx        <-- PRESENT
GAMTrace-GAMRemote User Response:{"...valid user data..."}

Silent Skip (token_type = ""):
GAMTrace-GAMRemote Token Response:{"access_token":"xxx","token_type":"",...}
																															 <-- ABSENT (no AddHeader line!)
GAMTrace-GAMRemote User Response:{"Code":114,"Message":"Token no valido..."}
```

### Detection Rule
When you see a Token Response followed by a Code 114 error on userinfo, and there is no `GAMTrace-GAMRemote AddHeader: Authorization:` line between them, the `token_type` was empty and the Authorization header was silently skipped

---

## Sub-IDP Escalation (Client with Daughter Sessions)
Verified Pattern: When a GAM client receives a signout (`server_ip=0`) but has active daughter sessions (it acts as IDP for other clients), it escalates its role from client to sub-IDP within the same request. This is NOT a separate redirect — it happens inline

### Behavioral Flow
```
1. SLO processing begins with server_ip=0 (client role)
2. Token to finish is located in the session store
3. Session is found with no parent token (root session)
4. GAM checks: does this session have child sessions?
	 - YES (has daughters):
		 a. Save IDP and Client state snapshots
		 b. Escalate to sub-IDP role (observable as server_ip changing from 0 to 2)
		 c. Kill own session
		 d. Fall through to IDP processing path (server_ip >= 1)
	 - NO (no daughters):
		 Standard client SLO (no escalation)
```

### Trace Progression
```
GAMTrace-SLOProcess - Parm_isServerIP:0   Parm_FirstCall:0   <-- Enters as client
GAMTrace-SLOProcess - IDP-IDP                                 <-- Escalation triggered
GAMTrace-SLOProcess - Parm_isServerIP:2   Parm_FirstCall:1   <-- Now acting as sub-IDP
GAMTrace-SLOProcess - &SesAppId:N  &Token:...  &haveChildren:true
GAMTrace-SLOProcess - IDP - SLO process initiated in IDP - &ParentToken:...
GAMTrace-SLOProcess -  ==IDP-EndDaughtersSessions ++++++      <-- Processing daughters
```

### Key Insight
After escalation, server_ip=2 causes the flow to enter the IDP-of-IDP path. The escalated client processes its daughter sessions exactly like a real IDP would

---

## Session End Date as SLO Notification Marker
Verified Pattern: GAM uses a two-phase approach for multi-daughter SLO:

- Phase 1 (Bulk mark): Set status to finished (no end date)
	* Session Status: `F`
	* End Date: Empty / default
	* Meaning: Killed in DB, pending notification
- Phase 2 (Process one): Set end date to current time
	* Session Status: `F`
	* End Date: Actual timestamp
	* Meaning: Killed in DB, notification attempted

### Why This Works
- The "find next session to finish" query looks for: status=`F` AND end date is empty — finds only unprocessed daughters
- Each daughter gets its end date set BEFORE the redirect — it will not be found again even if the redirect fails
- The loop continues when a daughter returns: callback → find next unprocessed daughter → process it
- When no more daughters remain: redirect back to initial SLO origin

---

## SLO URL Matching Algorithm
This algorithm determines which custom SLO URL to use when a non-GAM application has multiple deployment URLs registered

### Algorithm Behavior
```
Input:
	CustomURLsList = "http://host1/app1/slo;http://host2/app2/slo;..."
	SessionURL = The callback URL stored in the session at login time

If multiple URLs in the list:
	1. Progressively shorten the session URL from right to left (removing path segments)
	2. At each level, compare against each custom URL in list order
	3. First match wins

If only one URL in the list:
	Return it directly

If no match:
	Log error, return empty
```

### Key Properties
- Most-specific-first: Matches at app-path level before domain level
- Order-dependent: Within the same matching level, the first URL in the list wins
- Case-insensitive: Both URLs are lowercased before comparison
- Max 20 iterations: Safety limit on path depth
- Session URL is the anchor: The session's callback URL (stored at login time) determines which deployment is active

### Common Diagnostic Pattern
```
GAMTrace-Parm2:http://callback.com/slo;http://host/ppp;http://host/AppNetSQL/slo.aspx
GAMTrace-Parm3:http://host/AppNetCoreSQL/asso_callback.aspx
GAMTrace-&AppCliSLOURL:http://host/ppp
```
Diagnosis: No URL in the list matches at the app-path level (`AppNetCoreSQL`). Falls to domain level and selects `http://host/ppp` (first domain match). Fix: add `http://host/AppNetCoreSQL/slo` to the list

---

## Non-GAM Client Authentication Pattern
A GeneXus KB without GAM can authenticate against a GAM IDP using standard OAuth 2.0 Authorization Code flow

### Login Flow
```
1. Client stores OAuth config (IDP URL, client_id, client_secret, PKCE params) in web session
2. Client redirects browser to: {IDPURL}/oauth/gam/signin?scope=...&client_id=...&redirect_uri=...&response_type=code
3. User authenticates at GAM IDP → IDP redirects to client's callback URL with auth code
4. Client's callback exchanges code for token:
	 - POST to {IDPURL}/oauth/gam/access_token → receives access_token, token_type, user_guid
	 - GET to {IDPURL}/oauth/gam/userinfo with Authorization header → receives user info
5. Client stores token and user info in web session
```

### Logout Flow (Non-GAM Client Initiating)
Two patterns exist for the client-initiated logout:

Pattern A — Redirect-based (with SLO chain support):
Build a signout URL: `{IDPURL}/oauth/gam/signout?client_id={ClientId}&redirect_uri={RedirURL}&token={access_token}`, clear the local session, and redirect the browser. This triggers the full SLO chain at the IDP

Pattern B — REST call (no SLO chain):
Call the signout endpoint via HTTP GET from server-side code. This kills the token but does NOT trigger the SLO chain to other clients because there is no browser redirect

### Key Configuration for Non-GAM Apps in GAM Backoffice
- `ClientCallbackURL`: Custom URL (e.g., `http://host/app/asso_callback.aspx`)
	* Why: Non-standard callback, not `/oauth/gam/callback`
- `ClientCallbackURLisCustom`: `true`
	* Why: Tells GAM not to append `/oauth/gam/callback`
- `ClientSingleLogoutDisableSLO`: `false`
	* Why: Enable SLO for this app
- `ClientSingleLogoutCustomURLsSLO`: Semicolon-separated SLO handler URLs
	* Why: One per deployment; GAM uses the URL matching algorithm (see "SLO URL Matching Algorithm" above) to select
