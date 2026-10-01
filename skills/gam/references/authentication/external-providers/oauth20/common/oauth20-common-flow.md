---
name: oauth20-common-flow
description: Observable OAuth 2.0 Authorization Code flow when GAM is the client (SP) against any external OAuth 2.0 IDP — trace signatures, state lifecycle, token exchange, UserInfo mapping, SLO return trip. Shared by every provider file
---

# GAM-as-SP — OAuth 2.0 Common Flow (observable in trace)
Scope: provider-agnostic behavior of the OAuth 2.0 Authorization Code flow when GAM acts as the OAuth client against any external IDP (Okta, Keycloak, Auth0, Google, Microsoft Entra ID, Facebook, LinkedIn, Apple, AGESIC, custom). The sibling `../external-providers/` folder covers per-IDP configuration; this file covers the observable flow. Use it to read logs, identify the phase, and locate misconfiguration

Everything below is observable via enabled tracing and the Backoffice — no internal source required. Signatures use the v18u15+ structured-JSON log format (`genexus.security.api.<ProcName> - <Phase> - {"data":{…}}`), which is cross-generator (.NET, .NET Framework, .NET Core, Java). Where a signature predates v18u15, the legacy `GAMTrace-` form is listed alongside

Related files:
- [OAuth 2.0 Auth Type — Programmatic Initialization](../provider-generic-oauth20.md) — configuration template for any OAuth 2.0 IDP
- [GAM Authentication](../../common/domain-auth.md) — top-level login flows and scopes
- [GAM Authentication Configuration Reference](../../common/domain-auth-config.md) — `AuthenticationOAuth20SDT` property tree
- [GAM External Authentication Entry Point](../../common/domain-extauthinput.md) — `OriginType` / `OriginTypeValue` decision matrix
- [GAM Trace Signatures Catalog](../../../../debugging/trace-signatures-catalog.md) — full signature catalog (Format A + Format B)
- [GAM Trace Analyzer](../../../../debugging/common-trace-analyzer.md) — 5-pass methodology and format detection
- [GAM Cache Debugging](../../../../debugging/cache-debugging.md) — cache keys populated during OAuth flow
- [SLO External IDP Analysis Playbook](../../../../logout/domain-slo-external-idp.md) — SLO return trip through `/oauth/gam/callback`

---

## Observable Phases (happy path)
A successful OAuth 2.0 external login produces these phases in order. Each phase has a trace fingerprint and a Backoffice configuration that governs it

- Page render — GAM login page (`gamexamplelogin` or a KB-specific equivalent) draws the provider button
- Anonymous session bootstrap — GAM creates a temporary session and writes its token into the browser WebSession
- Authorize URL build — GAM composes the IDP's `authorize` URL with `response_type=code`, `client_id`, `scope`, `redirect_uri`, `state` (and `code_challenge` if PKCE)
- State persistence — GAM serializes its end-of-flow context and stores it under the `state` value; the state is one-time use
- Browser redirect to IDP — HTTP 302 to the IDP's authorize endpoint
- User authenticates at the IDP — opaque to GAM
- IDP callback — HTTP 302 to `/oauth/gam/callback?code=<code>&state=<state>…`
- Callback validation — GAM reads `state`, retrieves the persisted context (one-time, deleted on read), confirms match
- Token exchange — GAM server-to-server POST to the IDP `token` endpoint with `grant_type=authorization_code`, `code`, `client_id`, `client_secret` (or PKCE verifier), `redirect_uri`. Response carries `access_token`, `token_type`, `expires_in`, optional `refresh_token`, optional `id_token` (OIDC)
- UserInfo fetch — GAM sends `Authorization: Bearer <access_token>` to the IDP's `userinfo` endpoint. Response is a JSON object
- UserInfo mapping — GAM maps response fields into `GAMUser` via the `AuthenticationOAuth20SDT.UserInfo.Response*_Name` configuration
- User upsert — GAM creates or updates the `GAMUser` row keyed by the `ResponseUserExternalId_Name` claim; other fields refresh from the claims
- Event firing — `Repository_Login` and `User_Insert` / `User_Update` fire synchronously against subscribed listeners
- Authenticated session write — a fresh session token replaces the anonymous token and is persisted to the `gam.Session` table
- Post-login redirect — HTTP 302 to the URL stored under `FromURL` in the state context (typically the app's home)

If any phase is missing from a trace, the root cause is between that phase and the previous one. See "Diagnostics — What's Missing When Things Go Wrong" below for "what's missing" diagnostics

---

## Trace Signatures by Phase (v18u15+ Format B, with legacy Format A where relevant)
Each line below is the anchor you search for. The `{"data":{…}}` payload carries the live values referenced in later sections

### Anonymous session bootstrap (phase 2)
Format B:
- OK — `genexus.security.api.GAMSessionAPIRead - Start_Sub-SessionNew - {"data":{"SessionType":1}}`
- OK — `genexus.security.api.GAMSessionAPIRead - Start_Sub-SessionCreated - {"data":{"Token":"<RepoGUID>!<rand>"}}`
- OK — `genexus.security.api.GAMSaveWebSessionSession - End_Method - {"data":{"Parm2":true}}` — anonymous token written into the browser WebSession

Token shape: `<RepoGUID>!<base64-random>`. The `RepoGUID!` prefix identifies the repository for multi-tenant routing — see [GAM Multi-Tenant & Repository Encyclopedia](../../../../multi-tenant/domain-multitenant.md)

### Authorize URL build (phase 3)
Format B:
- OK — `genexus.security.api.GAMAuthenticationLogin - Start_Method - {"data":{"Parm1_AuthTypeName":"<auth-type>","Parm2_RepositoryGUID":"<GUID>"}}`
- OK — `genexus.security.api.GAMExternalAuthenticationOAuth20 - Start_Method - {"data":{"Parm1":1}}` — `Parm1=1` is the "initiate" branch (build authorize URL)
- OK — `genexus.security.api.GAMExternalAuthenticationOAuth20 - GoToIP-AuthType-OAuth20 - {"data":{"URL":"https://…authorize?…"}}`

The `URL` payload is the exact redirect the browser receives. Missing `state`, missing `client_id`, or wrong `redirect_uri` here means the corresponding `AuthenticationOAuth20SDT.Authorize.*_Include` flag is False or the `*_Value` property is empty — see [GAM Authentication Configuration Reference](../../common/domain-auth-config.md)

### State persistence (phase 4)
Format B:
- OK — `genexus.security.api.GAMGenerateToken - End_Method - {"data":{"Parm1":2,"Parm2":40,"retval":"<40-char-random>"}}` — 40-char random; state becomes `OA2STD<random>` for external OAuth 2.0 login, `SLOExt<random>` for external SLO
- OK — `genexus.security.api.GAMStateClientAPI - Start_Method - {"data":{"Parm1":"set","Parm2":{"GAMTokenState":"<state>","RepositoryGUID":"<GUID>","ApplicationId":<N>,"AuthenticationTypeName":"<auth-type>","FromURL":"<return-URL>","DeviceId":"<id>"}}}`

Prefix legend: `OA2STD` = OAuth 2.0 external login state; `SLOExt` = external SLO state; `SLOInt` = internal SLO state; `GRESTD` = GAMRemote delegation. If the prefix doesn't match the flow you expect, you're on a different branch — see [GAM Behavioral Patterns](../../../../debugging/common-behavioral-patterns.md)

The `FromURL` field determines where the user lands after phase 15. If it's empty or URL-encoded, the final redirect will break — see "Login succeeds but user lands on the wrong URL" below

### Callback validation (phase 8)
Format B:
- OK — `genexus.security.api.GAMExternalAuthenticationInputLoadParam - End_Method - {"data":{"Parm_State":"<state>","Parm_AccessCode":"<code>","Parm_OriginType":""}}` — `OriginType:""` is expected on external-IDP return; GAM falls into its "Otherwise" branch on purpose
- OK — `genexus.security.api.GAMStateClientAPI - Start_Method - {"data":{"Parm1":"gem","Parm2_State":"<state>"}}` — `gem` = get-and-remove; state is deleted from cache on this call, so replay is impossible
- OK — `genexus.security.api.GAMExternalAuthenticationInputValidParam - End_Method - {"data":{"isOK":true}}`

If `GAMStateClientAPI - Start_Method` with `Parm1:"gem"` returns `retval.GAMTokenState:""` or `isOK:false`, the state is either expired (TTL elapsed), already consumed (double callback), or forged (CSRF attempt). Symptom: user bounces back to login with no explicit error

### Token exchange (phase 9)
Format B:
- OK — `genexus.security.api.GAMExternalAuthenticationOAuth20 - Start_Method - {"data":{"Parm1":3}}` — `Parm1=3` is the "return from IDP" branch (POST to `token`, GET `userinfo`)
- OK — `genexus.security.api.GAMExternalAuthenticationOAuth20 - ReturnFromIP-Add-Header-AuthType-OAuth20 - {"data":{…}}`
- OK — `genexus.security.api.GAMExternalAuthenticationOAuth20 - ReturnFromIP-Add-GrantType-AuthType-OAuth20 - {"data":{"GrantType":"authorization_code"}}`
- OK — `genexus.security.api.GAMSearchJsonLabel - End_Method - {"data":{"Label":"access_token","retval":"<token>"}}`
- OK — `genexus.security.api.GAMSearchJsonLabel - End_Method - {"data":{"Label":"expires_in","retval":"<seconds>"}}`
- OK — `genexus.security.api.GAMSearchJsonLabel - End_Method - {"data":{"Label":"refresh_token","retval":"<token>"}}` — empty string if IDP did not issue one (e.g., `offline_access` scope not requested)

`GAMSearchJsonLabel` runs once per JSON field extracted. Counting its invocations tells you how many claims were read from token + UserInfo responses

### UserInfo fetch and mapping (phases 10–11)
Format B:
- OK — `genexus.security.api.GAMExternalAuthenticationOAuth20 - ReturnFromIP-UserInfo-AuthType-OAuth20 - {"data":{…}}`
- OK — `genexus.security.api.GAMSearchJsonLabel - End_Method - {"data":{"Label":"<ResponseUser*_Name>","retval":"<value>"}}` — one line per claim mapped
- OK — `genexus.security.api.GAMExternalAuthenticationGetDynAtt - End_Method - {"data":{"Attributes":[{"Id":"<claim>","Value":"<value>"}…]}}` — claims not in the standard map are stored as dynamic attributes

Claim-to-field mapping is driven entirely by `AuthenticationOAuth20SDT.UserInfo.Response*_Name` (see [GAM Authentication Configuration Reference](../../common/domain-auth-config.md)). The external identifier stored in `GAMUser.UserExtId` comes from the `ResponseUserExternalId_Name` claim — if that claim is misspelled or missing from the IDP response, every login creates a new user instead of updating the existing one

### User upsert and events (phases 12–13)
Format B:
- OK — `genexus.security.api.GAMUpdateOrCreateUserInGAM - Start_Method - {"data":{"User":{"UserExtId":"<id>","UserEMail":"<email>","UserFirstName":"<name>","UserLastName":"<name>"…}}}`
- OK — `genexus.security.api.ExecuteEventSubscriptions - Start_Method - {"data":{"EventName":"repository-login"}}`
- OK — `genexus.security.api.ExecuteEventSubscriptions - Start_Method - {"data":{"EventName":"user-insert"}}` — fires only on first-time login; subsequent logins fire `user-update`
- OK — `genexus.security.api.ExecuteEventSubscriptions - Start_Sub-ExecuteExternalObject - {"data":{"FileName":"<listener>.dll","ClassName":"<ns>.<class>","MethodName":"execute"}}`

Events fire synchronously before the session is persisted. A slow or hung listener blocks the rest of the login; see [GAM Events Subscription: Handler Contract](../../../../events/domain-event-subscription.md) and [GAMEventSubscription Code Provider](../../../../kb-setup/init/code-provider/gameventsubscription-code-provider.md)

### Session write and post-login redirect (phases 14–15)
Format B:
- OK — `genexus.security.api.GAMGenerateToken - End_Method - {"data":{"Parm1":2,…}}` — fresh authenticated token
- OK — `genexus.security.api.GAMSaveWebSessionSession - End_Method - {"data":{"Parm2":true}}` — replaces the anonymous token
- OK — `genexus.security.api.GAMSaveSessionToDB - End_Method - {"data":{"isOK":true}}` — row inserted in `gam.Session` with Status='A'
- OK — `genexus.security.api.ChangeURLToCurrentVirtualDir - End_Method - {"data":{"retval":"<post-login-URL>"}}`

---

## State Parameter Lifecycle (CSRF protection)
The `state` parameter is a 40-char random token prefixed per flow (`OA2STD…` for external OAuth 2.0 login). It is:

- Generated — `GAMGenerateToken` with `Parm1=2, Parm2=40`
- Serialized with context (repository GUID, application id, auth type name, `FromURL`, device id) into a JSON SDT (observable as `Parm2` payload in the `GAMStateClientAPI - "set"` trace line)
- Persisted — server-side, keyed by the state value, with a TTL tracked internally (the cache key `com.genexus.gam.general:LastRunDemon` records the last cleanup run)
- Returned to the IDP in the authorize redirect
- Received back on `/oauth/gam/callback`
- Retrieved and deleted atomically — `GAMStateClientAPI` with `Parm1="gem"` (get-and-remove) — so replay is impossible
- Compared to the inbound `state` query parameter — mismatch raises a validation error; match unwraps the stored context

Failure modes tied to state:
- State not returned by IDP → IDP stripped it (non-compliant) or the authorize URL never included it (check `Authorize.State_Include`)
- State retrieved but `isOK:false` → context expired; user waited too long at the IDP
- State already consumed → double callback (browser back button, repeated redirect); user bounces to login
- `FromURL` stored URL-encoded (Java-generator historical bug) → final redirect malformed — see [GAM Cross-Generator Issues Encyclopedia](../../../../environments/domain-cross-generator.md)

---

## Cache Keys Populated During the Flow
Observable via `genexus.security.api.GAMGetCache` and `GAMSetCache` trace lines. See [GAM Cache Debugging](../../../../debugging/cache-debugging.md) for eviction rules

- `com.genexus.gam.connectionfile` keyed by connection name — DB connection block; first read on page load, cached thereafter
- `com.genexus.gam.repositories` keyed by `<RepositoryGUID>` — full repository metadata; first read on page load
- `com.genexus.gam.applicationfile` keyed by `<AppName>` — application block
- `com.genexus.gam.applications.repository_<RepoGUID>` with sub-keys `AppGUID:<GUID>-` and `AppCliId:<ClientID>-`
- `com.genexus.gam.general` with key `GeneralSettings` (multi-tenant flag, tracing flag, OIDC issuer, KeysExpirationTime) and key `LastRunDemon` (state-cache cleanup bookkeeping)
- `com.genexus.gam.eventsubs.repository_<RepoGUID>` keyed by `Event:<eventName>-` — subscriber list for each event; `Cache-NotFound` here is benign (no subscribers)
- `com.genexus.gam.apppermission.repository_<RepoGUID>` keyed by `AppId:<id>-Prm:<perm>` — permission-check cache

None of these caches are purged during an OAuth login or SLO. Configuration changes made via Backoffice while a user is mid-session are NOT picked up until the cache TTL expires or the app restarts

---

## Diagnostics — What's Missing When Things Go Wrong
Use the phase anchors in "Trace Signatures by Phase" above. The rule: if phase N is missing but phase N-1 is present, the defect lies between them

### No authorize redirect (phases 1–3 present, nothing after)
Symptom: login page stays open, provider button does nothing, or browser lands on an error page with no GAM log line after phase 2

Likely causes:
- `AuthenticationOAuth20SDT.IsEnable = False` — auth type disabled in Backoffice
- `RedirectToAuthenticate = False` — GAM expects the caller to post credentials locally
- Discovery URL unreachable when `OpenIDConnectAuthentication.UseDiscoveryURL = True` — network path from the GAM server to the IDP is blocked

### Browser lands on `/oauth/gam/callback` but callback validation fails
Symptom: `GAMExternalAuthenticationInputLoadParam` line present, but no subsequent `GAMStateClientAPI - "gem"` Start or the Start returns `isOK:false`

Likely causes:
- State expired (user paused at IDP consent screen longer than the state TTL)
- State double-consumed (user hit browser back, then forward; or a second tab intercepted the callback)
- Mismatched repository — callback reached the wrong GAM namespace; check `ApplicationId` and `RepositoryGUID` in the state payload against the current request

### Token exchange fails
Symptom: `GAMExternalAuthenticationOAuth20 - Start_Method - Parm1:3` present, but no `GAMSearchJsonLabel - Label:"access_token"` line

Likely causes:
- IDP returned an error body; check for `GAMSearchJsonLabel - Label:"error_description"` with a non-empty `retval`
- `Token.URL` points to the authorize endpoint instead of the token endpoint
- `Token.Header_AuthorizationBasic_Include` wrong for the IDP (some require Basic, others require `client_secret` in body)
- PKCE required by the IDP but `PKCEAuthentication.Enable = False` — IDP returns `invalid_grant`

### Login succeeds but user lands on the wrong URL
Symptom: all session write lines present, but `ChangeURLToCurrentVirtualDir` retval is blank, localhost, or has `%3A%2F%2F` encoding inside

Likely causes:
- `FromURL` never captured when the login was initiated — the entry page didn't pass a return URL
- `FromURL` stored URL-encoded by a Java-generator version with the historical encoding bug; the downstream concatenation treats it as a path fragment
- `RedirectURL_AutocompleteVirtualDirectory = False` combined with a relative path

### Every login creates a duplicate user
Symptom: `GAMUpdateOrCreateUserInGAM` always takes the "create" branch; `GAMUser` count grows on every login

Likely causes:
- `ResponseUserExternalId_Name` is wrong for the IDP (e.g. set to `id` but IDP returns `sub`, or vice versa)
- IDP returns the external id under a nested JSON path that `GAMSearchJsonLabel` can't flatten
- `UserExtId` column truncation at the DB — the external id exceeds the defined length; upsert lookup never matches

### First login succeeds, second login fails with "session expired"
Symptom: `Repository_Login` event fires, `GAMSaveSessionToDB` writes a row, but the next request sees `Session expired for user`

Likely causes:
- Synchronous event listener is writing back into the session in a way that invalidates the fresh token — see [GAMEventSubscription Code Provider](../../../../kb-setup/init/code-provider/gameventsubscription-code-provider.md)
- Web server session timeout is shorter than `OauthTokenExpire` (Timeout Conflict) — see [GAM Security Policies Encyclopedia](../../../../authorization/domain-security-policies.md)
- Load balancer affinity not set; the second request lands on a node whose in-process cache doesn't know the session

---

## Cross-generator Notes
Everything under `genexus.security.api.*` is cross-generator. Lines under `GeneXus.*` (e.g., `GeneXus.Data.NTier.DataStoreProvider`, `GeneXus.Http.GxWebSession`, `GeneXus.Cache.InProcessCache`) are .NET-family-only stack detail and should not be relied on when supporting a Java deployment. The Java equivalents emit the same `genexus.security.api.*` signatures but different surrounding stack frames

Known Java-only quirks (see [GAM Cross-Generator Issues Encyclopedia](../../../../environments/domain-cross-generator.md) for the full list):
- `FromURL` stored URL-encoded in the state context in historical versions; final redirect double-encodes
- `GAMSearchJsonLabel` handling of deeply nested JSON paths

---

## Per-Provider Overrides
Every `provider-*.md` in `../external-providers/` covers:
- Endpoint URLs for that IDP
- Required `scope` values
- Canonical claim names for the `Response*_Name` mappings
- Signout endpoint and required parameters
- PKCE / OIDC quirks

Phase-level behavior (phases 1–15 above) is identical across all providers. If a provider trace diverges from "Trace Signatures by Phase" in a way not explained by "Diagnostics — What's Missing When Things Go Wrong", it's a provider quirk that belongs in that provider's file, not here

---

## Related SLO flow
The SLO counterpart (GAM-as-SP signing the user out of an external IDP) shares the same `state` machinery and callback path. Anchor procedures:

- `Logout_internal` — orchestrator, equivalent role to `GAMAuthenticationLogin` on the login side
- `GAMExternalAuthenticationSignout` — builds the external signout redirect with `state` (prefix `SLOExt`) and `post_logout_redirect_uri = <GAM-app>/oauth/gam/callback`
- On return, the callback arrives at `/oauth/gam/callback` with only `state` in the query string (no `code`, no `OriginType`); the same `GAMExternalAuthenticationInputValidParam` Otherwise branch handles it

Full SLO coverage: [GAM Single Log Out (SLO)](../../../../logout/domain-slo.md) and [SLO External IDP Analysis Playbook](../../../../logout/domain-slo-external-idp.md)
