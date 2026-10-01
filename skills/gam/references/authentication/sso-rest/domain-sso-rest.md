---
name: domain-sso-rest
description: SSO REST — Single Sign-On for REST Services using GAM; token format, 3-actor architecture, IDP and Client B configuration, end-to-end flow, trace patterns, and error catalog
---

# SSO REST — Single Sign-On for REST Services
Centralizes authentication across distributed REST services without browser redirects. Client A and Client B never need to know each other; they only share trust in a common GAM IDP. Client A authenticates against the IDP using GAMRemote (web) or GAMRemoteREST (REST) and receives an SSO REST token alongside the standard access token. When Client A calls a service on Client B, it passes this token; Client B validates it with the IDP server-to-server and creates its own local GAM session

Available since GeneXus 17

Related files:
- [GAM as SP — GAMRemote (Web Flow) Initialization](../local-sp/domain-gamremote.md) — Client A-side: Web Authorization Code flow; IDP emits SSO REST token when `SSORESTEnable=True, SSORESTMode=Server`
- [GAM as SP — GAMRemoteRest (REST Password Grant) Initialization](../local-sp/domain-gamremoterest.md) — Client A-side: REST Password Grant flow; same condition
- [GAM Backoffice — Applications (GAM_Applications)](../../backoffice/applications.md) — full Application property mapping for SSO REST section
- [GAM Backoffice — Repository (GAM_Repository)](../../backoffice/repository.md) — `EnableSSORESTAccessForUndefinedClientIDs` (Repository > Advanced)
- [GAM Single Log Out (SLO)](../../logout/domain-slo.md) — SLO interaction with `SSORESTServerURL_SLO`
- [GAM Trace Analyzer](../../debugging/common-trace-analyzer.md) — `TokenSSORest` field in `GAMSessionJSON_SDT` (v18u15+)
- [Official wiki](https://docs.genexus.com/en/wiki?46492,Single+Sign+On+for+Rest+Services+using+GAM) — overview, server-side and client-side configuration pages

---

## Token format
The SSO REST token is an opaque composite string — NOT a JWT, NOT cryptographically signed:

```
Rep_GUID!<TOKEN>@SSORT!Rep_GUID!Client_ID
```

- `Rep_GUID` — GUID of the IDP repository where the parent session lives
- `<TOKEN>` — standard GAM token (opaque hash)
- `@SSORT!` — literal delimiter; its presence identifies the token as SSO REST
- `Client_ID` — Application Client ID of Client A (the consumer)

Integrity validation is server-to-server only: Client B sends the full token to the IDP `/requesttokenanduserinfo` endpoint; the IDP looks up the parent session and compares `ParentSession.SSORestToken`. If they match, the token is valid

The `@SSORT!` delimiter is detected by `GAMSpecialTokenIsSSORest` (internal proc) on both Client A (response parsing) and Client B (incoming request detection)

---

## Architecture
Three actors; all three KBs require GAM activation:

- **IDP** — emits the SSO REST token when Client A authenticates (Application must have `SSORESTEnable=True, SSORESTMode=Server`); validates tokens via `/oauth/gam/v2.0/requesttokenanduserinfo` when Client B calls
- **Client A** — authenticates with the IDP using GAMRemote or GAMRemoteREST; receives the SSO REST token in its local GAM session at `&GAMSession.SSORestToken`; sends the token to Client B in the `Authorization` header along with `Client_id`
- **Client B** — receives the token; detects `@SSORT!`; calls the IDP to validate; creates its own local GAM session under the user's identity

Client A and Client B share no direct trust — both independently trust the same IDP

---

## End-to-end flow
- Client A authenticates with the IDP via GAMRemote (Authorization Code) or GAMRemoteREST (Password Grant), passing `repository_ssorest=<RepoGUID_of_ClientA>` in the token request
- IDP Application has `SSORESTEnable=True, SSORESTMode=Server` → IDP emits token in SSO REST format (`Rep_GUID!TOKEN@SSORT!Rep_GUID!ClientId`) alongside the standard access token
- Client A reads the token: `&GAMSession.SSORestToken` is populated in the local GAM session after login
- Client A calls a service on Client B, adding headers:
	* `Authorization: Bearer <ssorest_token>`
	* `Client_id: <ClientA_ClientId>`
	* `Repository: <IDP_RepoGUID>`
- Client B receives the request; `GAMSpecialTokenIsSSORest` (internal) detects `@SSORT!` and extracts parts
- Client B calls `<SSORESTServerURL>/oauth/gam/v2.0/requesttokenanduserinfo` (GET) on the IDP with:
	* Header `Authorization: Bearer <token>`
	* Header `Client_id: <ClientB_ClientId>`
- IDP validates: token format → Application `SSORESTEnable=True, Mode=Server` for ClientB's ClientId → parent session exists → `ParentSession.SSORestToken` matches the received token
- IDP returns a `RequestTokenAndUserInfoSDT` payload:
	* `token` block: access_token, token_type=Bearer, expires_in, refresh_token, scope, user_guid
	* `user` block: claims scoped to `GAMApplicationValidScopes` (the scopes Client B is allowed to see)
- Client B creates its local GAM session and processes the request under the authenticated user's identity

---

## Configuration: IDP-side (Application in the IDP KB)
Properties of the Application that Client A authenticates against:

- `SSORESTEnable = True` — enables SSO REST token emission for this Application; without this the IDP returns a standard token only
- `SSORESTMode = Server` — identifies this Application as the SSO REST server; required for `/requesttokenanduserinfo` to accept validation calls for this ClientId
- `SSORESTUserAuthenticationTypeName` — name of the authentication type that governs which users can receive SSO REST tokens; must exist in the IDP repository

Repository-level (Backoffice: Repository > Advanced):
- `EnableSSORESTAccessForUndefinedClientIDs = True` — allows the IDP to validate SSO REST tokens for ClientIds not explicitly registered in its Application table; required when Client B's ClientId is not pre-registered at the IDP

Backoffice path (IDP): Applications → select application → Configuration > SSO REST

---

## Configuration: Client B-side (Application in the KB that exposes services)
- `SSORESTEnable = True` — activates SSO REST validation for incoming requests on this Application
- `SSORESTMode = Client` — marks this Application as a consumer; triggers server-to-server validation against the IDP on every incoming SSO REST request
- `SSORESTServerURL` — base URL of the IDP (e.g., `https://idp-server/idp-virtual-dir/`); Client B appends `/oauth/gam/v2.0/requesttokenanduserinfo` automatically unless `SSORESTServerURL_isCustom = True`
- `SSORESTServerURL_isCustom` — `True`: uses the URL value literally (must include the full path); `False` (DEFAULT): GAM appends the endpoint path automatically
- `SSORESTServerURL_SLO` — IDP URL used specifically for the SLO logout flow; may differ from the runtime URL; see [GAM Single Log Out (SLO)](../../logout/domain-slo.md)
- `SSORESTServerRepositoryGUID` — GUID of the IDP repository; scopes validation to the correct multi-tenant repository; must match the IDP repository's actual GUID
- `SSORESTServerKey` — symmetric key (Encrypt64/Decrypt64) used when the IDP Application has `RemoteServerKey` configured; must match byte-for-byte

Also required on Client B Application (Configuration > REST):
- `ClientAllowRemoteRESTAuthentication = True` — Client B must allow REST authentication to call the IDP endpoint; without this: `SSORestMustActivateRESTAuthentication`

Backoffice path (Client B): Applications → select application → Configuration > SSO REST

---

## Critical invariant: Client_id must match
The `Client_id` of the Application as registered in the **IDP** MUST match the `Client_id` of the same Application registered in **Client B**. If they differ:

- Error: `SSORestNotEnabledOrServerModeIDPServer` (code 561)
- The IDP lookup fails because it cannot find an Application record matching the presented ClientId

This is the most frequent SSO REST configuration error. Verify both values in Backoffice (IDP and Client B) before debugging further

---

## Client A — retrieving and sending the token
After a successful GAMRemote or GAMRemoteREST login, the SSO REST token is available on the local session:

```genexus
&GAMSession = GAMSession.Get(&GAMErrorCollection)
If not &GAMSession.SSORestToken.IsEmpty()
	&httpClient.AddHeader(!"Authorization", &GAMSession.SSORestToken)
	&httpClient.AddHeader(!"Client_id", !"<ClientA_ClientId>")
	&httpClient.Execute(HttpMethod.Post, &StrCall)
EndIf
```

If `SSORestToken` is empty after a successful login: the IDP Application does not have `SSORESTEnable=True, SSORESTMode=Server` — or the Application ClientId used by Client A is not the one configured with SSO REST on the IDP

---

## Internal code paths (for trace correlation — do not expose to user)
Internal GAM procedures that implement SSO REST — use these to correlate trace messages with flow phases:

- `GAMExternalAuthenticationGAMRemote` — runs on Client A during browser (Authorization Code) login; passes `repository_ssorest` to the IDP in the redirect URL; does not generate the SSO REST token itself — the IDP does
- `GAMExternalAuthenticationGAMRemoteRest` — runs on Client A during REST (Password Grant) login; calls `GAMSpecialTokenIsSSORest` to detect the SSO REST token in the IDP's `access_token` response field; adds `Client_id` header to the UserInfo call when SSO REST is active
- `GAMSSORestRequestTokenAndUserInfo_v20` (HTTP GET, IsMain=True) — IDP endpoint at `/oauth/gam/v2.0/requesttokenanduserinfo`; validates Application `SSORESTEnable+Mode=Server`, parent session existence, and token match; returns user info and a fresh token to Client B
- `GAMSSORestValidClientApplicationAccessToken` — runs on Client B; sub-routine `ValidTokenServerSSORest` builds the URL from `SSORESTServerURL` and executes the GET to the IDP
- `GAMSpecialTokenIsSSORest` — parser; detects `@SSORT!` delimiter and splits the token into components

---

## Trace patterns
### Client A — GAMRemoteREST login (Format A, v18u14-)
Sequence in Client A log:
- `GAMTrace-==+++GAMExternalAuthenticationGAMRemoteRest ====== START ---`
- `GAMTrace-GAMRemote Token URL: <url>` — POST to IDP `/oauth/gam/v2.0/access_token`
- `GAMTrace-GAMRemote Token Response: <json>` — if SSO REST active, `access_token` value contains `@SSORT!`
- `GAMTrace-GAMRemote AddHeader: Client_id` — **present only when SSO REST token detected in response**
- `GAMTrace-GAMRemote User URL: <url>` — GET to IDP `/oauth/gam/v2.0/userinfo`

Absence of `AddHeader: Client_id` after `Token Response` → token is not SSO REST format → IDP Application does not have `SSORESTEnable=True, SSORESTMode=Server`

### Client A — GAMRemoteREST login (Format B, v18u15+)
- `DEBUG genexus.security.api.GAMExternalAuthenticationGAMRemoteRest - Start_Method - {"data":{…}}`
- `DEBUG genexus.security.api.… - End_Method - {"data":{"GAMSessionJSON_SDT":{"TokenSSORest":"SSORT!…",…}}}` — `TokenSSORest` populated = SSO REST active; empty string = not active

### IDP endpoint — requesttokenanduserinfo (Format A, v18u14-)
In IDP log when Client B calls to validate:
- `GAMTrace-==+++GAMSSORestRequestTokenAndUserInfo_v20 ====== START ---`
- `GAMTrace-Method: GET` — correct; any other HTTP method → error `ServiceMethodUnrecognized` (code 541)
- `GAMTrace-Access_token: <token>` — token received in Authorization header
- `GAMTrace-&isSSORest: True` — confirmed SSO REST format; `False` → format invalid
- `GAMTrace-&SSORestToken: <token>` — the stored token on the parent session (must match received token)
- `GAMTrace-&Response: <json>` — response payload on success
- `GAMTrace-&Errors: <json>` — errors array if validation failed

### Persistent log entries (application.gam.log — all format versions)
Messages written by `GAMGenerateGeneralLog` regardless of trace activation level:
- `"in the RequestToken and Userinfo service cannot read Authorization header"` — IDP received request but Authorization header is missing or unparseable
- `"SSO Rest Error 1 - StatusCode:<code>"` — Client B received an HTTP error calling the IDP endpoint
- `"GAMRemote Error 1"` / `"GAMRemote Error 2"` — Client A received an HTTP error calling IDP `/access_token` or `/userinfo`

---

## Error catalog
- `SSORestNotEnabledOrServerModeIDPServer` (code 561) — IDP Application does not have `SSORESTEnable=True` or `SSORESTMode=Server` for the presented ClientId; most commonly a Client_id mismatch between IDP and Client B
- `SSORestTokenFormatNotValid` — received token does not follow `Rep_GUID!TOKEN@SSORT!Rep_GUID!ClientId` format; token was not issued as SSO REST, was truncated, or was corrupted in transit
- `SSORestTokenNotValid` — format is valid but `ParentSession.SSORestToken` on the IDP does not match the received token; session may have expired, been revoked, or the token was replayed from a different session
- `SSORestNotEnabledServicePublished` — Client B Application has `SSORESTEnable=False` but received an SSO REST token; enable SSO REST (Mode=Client) on the Client B Application
- `SSORestTokenNotFoundInIDPResponse` — IDP `/requesttokenanduserinfo` responded HTTP 200 but the payload was empty or structurally invalid
- `SSORestAccessError` — Client B call to the IDP endpoint failed with an HTTP error; check StatusCode + ErrDescription in `application.gam.log`; common causes: wrong URL, network timeout, IDP not running
- `SSORestMissingAuthenticationType` — Client B Application has `SSORESTEnable=True` but `SSORESTUserAuthenticationTypeName` is empty
- `SSORestAuthenticationTypeNotFound` — `SSORESTUserAuthenticationTypeName` references a name that does not exist in the repository
- `SSORestAuthenticationTypeNotValid` — auth type exists but is not compatible with SSO REST
- `SSORestMissingServerURL` — Client B Application has `SSORESTEnable=True, SSORESTMode=Client` but `SSORESTServerURL` is empty
- `SSORestMustActivateRESTAuthentication` — Client B Application does not have `ClientAllowRemoteRESTAuthentication=True`; required for Client B to make REST calls to the IDP
- `AppTokenNotFound` (code 123) — the IDP has no session or Application record matching the ClientId extracted from the token; verify ClientId registration on both the IDP and Client B
- `AccessTokenNotFound` — Authorization header received but no token could be extracted
- `UserGUIDErrorTryingLogin` — user GUID embedded in the SSO REST token does not match the GUID returned by UserInfo on the IDP
- `GAMRemoteEncryptionKeyError` — the IDP encrypted transport parameters using `RemoteServerKey` but Client A or Client B does not have the matching key configured

---

## Common pitfalls
- **Client_id mismatch** — the same Application must have identical ClientId values in the IDP and in Client B; different KB generators may produce different default values; always verify in Backoffice on both sides before any other debugging step
- **`SSORESTServerURL_isCustom=True` with incomplete URL** — when `isCustom=True`, `SSORESTServerURL` must include the full endpoint path including `/oauth/gam/v2.0/requesttokenanduserinfo`; when `isCustom=False` (DEFAULT), GAM appends the path automatically — mixing these causes `SSORestAccessError` with a 404
- **Repository GUID mismatch** — `SSORESTServerRepositoryGUID` must match the actual GUID of the IDP repository; stale GUIDs after environment refresh or cloning cause `AppTokenNotFound`
- **`SSORESTServerKey` missing** — when the IDP Application has `RemoteServerKey` configured (encrypted transport), Client B must have the matching key in `SSORESTServerKey`; absence causes `GAMRemoteEncryptionKeyError`
- **Missing `EnableSSORESTAccessForUndefinedClientIDs`** — when Client B's ClientId is not explicitly registered in the IDP's Application table, the IDP repository must have this flag enabled; its absence causes `SSORestNotEnabledOrServerModeIDPServer` even with a correct Client_id
- **`ClientAllowRemoteRESTAuthentication=False` on Client B** — Client B must have REST authentication enabled in Application > Configuration > REST; without it, `SSORestMustActivateRESTAuthentication` fires before any token validation occurs

---

## Single Logout interaction
When SSO REST is active, logout must propagate to both Client A's and Client B's sessions

- `SSORESTServerURL_SLO` on the Client B Application specifies the IDP URL for SLO; may differ from the `SSORESTServerURL` used for runtime validation
- When Client A logs out and the IDP triggers the SLO chain, it must reach Client B; if Client B's session is not terminated, the user remains authenticated there
- The `token` parameter sent by the IDP to non-GAM SLO handlers is the SSO REST token — the handler must verify it matches the stored `access_token`; if SSORestToken is enabled the stored value is the SSO REST token, not the standard access token

See [GAM Single Log Out (SLO)](../../logout/domain-slo.md) for SLO phase flow, chaining, and the Non-GAM Client SLO section

---

## When to use SSO REST vs alternatives
- **vs API Key** — SSO REST propagates the authenticated end-user identity and scopes; API Key identifies the service application, not the user; use SSO REST when Client B needs to know WHO the user is
- **vs OAuth Client Credentials (machine-to-machine)** — Client Credentials grant identifies the service; SSO REST carries end-user identity from Client A to Client B across service boundaries; use SSO REST when user context must be preserved
- **vs manual token relay** — manual relay requires Client B to trust Client A's token directly (tight coupling); SSO REST uses canonical IDP-based validation with proper session lifecycle and scoped claims; prefer SSO REST for production inter-service scenarios

---

## See also
- [Official wiki — SSO REST overview](https://docs.genexus.com/en/wiki?46492,Single+Sign+On+for+Rest+Services+using+GAM)
- [Official wiki — Server-side configuration](https://docs.genexus.com/en/wiki?46496,Server-side+configuration+for+SSO+in+REST+applications)
- [Official wiki — Client-side configuration](https://docs.genexus.com/en/wiki?46499,Client-side+configuration+for+SSO+in+REST+applications)
