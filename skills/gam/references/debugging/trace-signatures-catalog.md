---
name: trace-signatures-catalog
description: Central catalog of all GAM trace signatures grouped by domain — use when analyzing any trace
---

# GAM Trace Signatures Catalog
Central reference for every observable trace signature. Load this file instead of pulling signatures from individual domain files when analyzing traces

Legend:
- `OK —` trace expected on healthy path
- `FAIL —` trace indicates failure or denial
- `<placeholder>` variable value in the actual trace

Format detection: Format A is legacy `GAMTrace-` prefix (up to v18u14). Format B is structured JSON `genexus.security.api.*` (v18u15+). See [GAM Trace Analyzer](common-trace-analyzer.md) for detection and 5-pass methodology

---

## Authentication (#auth)
### Login and OAuth flows
- OK — `GAMTrace-Login:<userName> - Success`
- OK — `GAMTrace-ValidateUserApp:<userGUID>`
- OK — `GAMTrace-GAMGetDefaultAuthType:<typeName>`
- OK — `GAMTrace-ExternalLogin - Authorization code received`
- FAIL — `GAMTrace-Login failed: <reason>`
- FAIL — `GAMTrace-Silent Skip detected — external auth type not invoked`

### Token exchange and validation
- OK — `GAMTrace-Session created Token=<token>`
- OK — `GAMTrace-Token refreshed for user <userId>`
- FAIL — `GAMTrace-Session Oauth, GeneXus.SD.Actions.Login is required to access`
- FAIL — `GAMTrace-InvalidToken — Token=<token>`

---

## Authorization (#authz)
### Permission evaluation
- OK — `GAMTrace-CheckPermission:<permissionName> = GRANTED`
- FAIL — `GAMTrace-CheckPermission:<permissionName> = DENIED`
- FAIL — `GAMTrace-CheckPermission: DENIED (explicit) <permissionName>`

### Role evaluation
- OK — `GAMTrace-User Main Role:<roleName>`
- OK — `GAMTrace-User App Roles:<roleList>`

### User state
- OK — `GAMTrace-UserBlock - UserIsBlk: True` — user locked after max failed attempts
- FAIL — `GAMTrace-Session expired for user <userId>`

### Application and Repository
- OK — `GAMTrace-RepositoryAPI valid access:<repoId>`
- OK — `GAMTrace-RepositoryGetMiniAppAccessToken`

---

## Session (#session)
### Session lifecycle
- OK — `GAMTrace-Application Session:<details>`
- OK — `GAMTrace-Session created Token=<token>`
- FAIL — `GAMTrace-Session Oauth, GeneXus.SD.Actions.Login is required to access`
- FAIL — `GAMTrace-Session expired for user <userId>`

### State persistence across redirects
- OK — `GAMTrace-GAMStateClientAPI - Delete State:<key>` — state cleanup after redirect
- FAIL — `PRIMARY KEY violation … gam.LoginTmp` — shared-DB collision without `IDP-` prefix

---

## SAML (#saml)
### Assertion and response
- OK — `GAMTrace-SAMLAssertion validated` — assertion signature OK
- OK — `GAMTrace-SAMLResponse - StatusCode: Success`
- FAIL — `GAMTrace-SAMLAssertion - InvalidSignature`
- FAIL — `GAMTrace-SAMLResponse - StatusCode: Responder`
- FAIL — `GAMTrace-SAMLAssertion - Expired`

### SAML SLO
- OK — `GAMTrace-SAMLogoutRequest sent to <entityId>`
- OK — `GAMTrace-SAMLogoutResponse - StatusCode: Success`
- FAIL — `GAMTrace-SAMLogoutResponse - StatusCode: PartialLogout`

### Certificates
- FAIL — `GAMTrace-SAMLCertificate - Expired`
- FAIL — `GAMTrace-SAMLCertificate - NotTrusted`

---

## SLO (#slo)
Flat lookup only — for per-signature "meaning / absence indicates" diagnostic reasoning see [domain-slo.md § TRACE SIGNATURES](../logout/domain-slo.md); for the `GAMLogoutType` state-progression decision tree see [GAM Behavioral Patterns](common-behavioral-patterns.md)

### 3-phase flow
- OK — `GAMTrace-SLOProcess - Phase: 1` — local session close
- OK — `GAMTrace-SLOProcess - Phase: 2` — child session(s) close
- OK — `GAMTrace-SLOProcess - Phase: 3` — external IDP call
- OK — `GAMTrace-SLOProcess - IDP - Finish token:<CHILD_TOKEN>`
- OK — `GAMTrace-SLOProcess - IDP - Finish parent token:<PARENT_TOKEN>`

### External IDP chaining
- OK — `GAMTrace-SLOExternal - Calling IDP: <url>`
- FAIL — `GAMTrace-SLOExternal - IDP call failed: <reason>`
- FAIL — `GAMTrace-SLOExternal - No ExternalToken available`

### LogoutType progression
- OK — `GAMTrace-GAMLogoutType = 0 → 1 → 2` — expected escalation
- FAIL — `GAMTrace-GAMLogoutType stuck at <value>` — broken SLO chain

---

## OTP / 2FA (#otp)
- OK — `GAMTrace-OTP - Code generated for <userId>`
- OK — `GAMTrace-OTP - Code verified for <userId>`
- FAIL — `GAMTrace-OTP - Code expired`
- FAIL — `GAMTrace-OTP - Invalid code`
- FAIL — `GAMTrace-OTP - Max retries exceeded`

---

## Multi-Tenant (#multitenant)
### Repository resolution
- OK — `GAMTrace-GAMGetRepositoryConnection - &CacheConnectionCli:{…}`
- OK — `GAMTrace-GAMGetCacheRepository - 1 &CacheRepository:<GUID>`
- FAIL — `GAMTrace-GAMGetRepositoryConnection - &Errors:[<error>]`

### Cache
- OK — `GAMTrace-GAMGetCache - Cache name:com.genexus.gam.repositories Key:<GUID>`
- OK — `GAMTrace-GAMSetCache - RepId:<namespace>: Key:<GUID> Value:{…}`

---

## External Auth Input (#extauthinput)
- OK — `GAMTrace-GAMExternalAuthenticationInput - OriginType:<type>`
- OK — `GAMTrace-GAMExternalAuthenticationInput - OriginTypeValue:<value>`
- FAIL — `GAMTrace-GAMExternalAuthenticationInput - Unknown OriginType`

---

## Impersonation (#impersonation)
- OK — `GAMTrace-Impersonate - Admin:<adminId> Target:<targetId>`
- OK — `GAMTrace-Impersonate - New session Token=<newToken>`
- FAIL — `GAMTrace-Impersonate - Admin lacks GAM_Impersonate permission`

---

## Error Codes (cross-reference)
When a trace contains an error code, resolve the meaning through these files:
- Authentication and session codes → [GAM Authentication — Error Code Catalog](../authentication/external-providers/common/error-codes-auth.md)
- Authorization codes (20, 30, 114, 17) → see [GAM Authorization — Permission Evaluation](../authorization/domain-authz.md)
- SAML codes → see [GAM SAML 2.0 Reference](../authentication/external-providers/saml20/common/domain-saml.md) error section

---

## v18u15+ Format B Signatures (structured JSON)
Format B uses `DEBUG genexus.security.api.<ProcName> - <Phase> - {"data":{<payload>}}`. Every `genexus.security.api.*` logger is cross-generator (`.NET`, `.NET Framework`, `.NET Core`, Java). Lines under `GeneXus.*` (for example `GeneXus.Data.NTier.DataStoreProvider`, `GeneXus.Http.GxWebSession`, `GeneXus.Cache.InProcessCache`, `GeneXus.Metadata.ClassLoader`) are `.NET`-family-only stack frames and should not be relied on when supporting Java deployments

Phase vocabulary used below:
- `Start_Method` / `End_Method` — procedure entry / exit
- `Start_Sub-<SubName>` — named sub-routine entry
- `<NamedEvent>` — flow checkpoint inside a procedure

Canonical flow definition for SIGNIN + SLO: see [GAM-as-SP — OAuth 2.0 Common Flow (observable in trace)](../authentication/external-providers/oauth20/common/oauth20-common-flow.md)

### Authentication — OAuth 2.0 external login, GAM-as-SP (#auth)
Anonymous session bootstrap and login entry:
- OK — `genexus.security.api.GAMSessionAPIRead - Start_Sub-SessionNew - {"data":{"SessionType":1}}` — anonymous session bootstrap
- OK — `genexus.security.api.GAMAuthenticationLogin - Start_Method - {"data":{"Parm1_AuthTypeName":"<type>","Parm2_RepositoryGUID":"<GUID>"}}` — login entry

Initiate branch — build authorize URL and persist state:
- OK — `genexus.security.api.GAMExternalAuthenticationOAuth20 - Start_Method - {"data":{"Parm1":1}}` — initiate branch
- OK — `genexus.security.api.GAMExternalAuthenticationOAuth20 - GoToIP-AuthType-OAuth20 - {"data":{"URL":"<authorize-url>"}}`
- OK — `genexus.security.api.GAMGenerateToken - End_Method - {"data":{"Parm1":2,"Parm2":40,"retval":"<random>"}}` — 40-char state token
- OK — `genexus.security.api.GAMStateClientAPI - Start_Method - {"data":{"Parm1":"set","Parm2":{…}}}` — persist state snapshot

Return-from-IDP branch — token exchange and UserInfo:
- OK — `genexus.security.api.GAMStateClientAPI - Start_Method - {"data":{"Parm1":"gem"}}` — get-and-remove state on callback
- OK — `genexus.security.api.GAMExternalAuthenticationOAuth20 - Start_Method - {"data":{"Parm1":3}}` — return-from-IDP branch
- OK — `genexus.security.api.GAMExternalAuthenticationOAuth20 - ReturnFromIP-Add-Header-AuthType-OAuth20`
- OK — `genexus.security.api.GAMExternalAuthenticationOAuth20 - ReturnFromIP-Add-GrantType-AuthType-OAuth20 - {"data":{"GrantType":"authorization_code"}}`
- OK — `genexus.security.api.GAMExternalAuthenticationOAuth20 - ReturnFromIP-UserInfo-AuthType-OAuth20`
- OK — `genexus.security.api.GAMSearchJsonLabel - End_Method - {"data":{"Label":"<claim>","retval":"<value>"}}` — one per JSON field extracted
- OK — `genexus.security.api.GAMExternalAuthenticationGetDynAtt - End_Method - {"data":{"Attributes":[…]}}` — non-standard claims

User upsert, event subscriptions and session save:
- OK — `genexus.security.api.GAMUpdateOrCreateUserInGAM - Start_Method - {"data":{"User":{…}}}` — upsert
- OK — `genexus.security.api.ExecuteEventSubscriptions - Start_Method - {"data":{"EventName":"repository-login"}}`
- OK — `genexus.security.api.ExecuteEventSubscriptions - Start_Method - {"data":{"EventName":"user-insert"}}` — first login
- OK — `genexus.security.api.ExecuteEventSubscriptions - Start_Method - {"data":{"EventName":"user-update"}}` — subsequent logins
- OK — `genexus.security.api.GAMSaveSessionToDB - End_Method - {"data":{"isOK":true}}`
- OK — `genexus.security.api.ChangeURLToCurrentVirtualDir - End_Method - {"data":{"retval":"<post-login-URL>"}}`

Failure signatures:
- FAIL — `genexus.security.api.GAMStateClientAPI - Start_Method - {"data":{"Parm1":"gem",…"retval":{"GAMTokenState":""}}}` — state expired or double-consumed
- FAIL — `genexus.security.api.GAMSearchJsonLabel - End_Method - {"data":{"Label":"error_description","retval":"<msg>"}}` — IDP error in token response

### SLO — external-IDP Single Logout, GAM-as-SP (#slo)
Orchestrator and external redirect:
- OK — `genexus.security.api.Logout_internal - Start_Method` — orchestrator entry
- OK — `genexus.security.api.Logout_internal - Start_Sub-LogoutOAuth20`
- OK — `genexus.security.api.GAMRemoteLogout - Start_Method` — kills parent and daughter sessions
- OK — `genexus.security.api.GAMExternalAuthenticationSignout - Start_Method` — builds external IDP signout redirect
- OK — `genexus.security.api.GAMStateClientAPI - End_Method - {"data":{"Parm1":"set","Parm2":{"LogoutType":3,"EndLogout":1,…}}}` — SLO state snapshot

Cleanup and event subscriptions:
- OK — `genexus.security.api.GAMDeleteWebSessionAndCookie - End_Method - {"data":{"Parm2":2}}` — WebSession cleanup at chain end
- OK — `genexus.security.api.ExecuteEventSubscriptions - Start_Method - {"data":{"EventName":"repository-logout"}}`

Return trip from external IDP:
- OK — `genexus.security.api.GAMExternalAuthenticationInputLoadParam - End_Method - {"data":{"Parm_State":"<SLOExt…>","Parm_OriginType":""}}` — `OriginType` empty is EXPECTED for external IDP callbacks
- OK — `genexus.security.api.GAMExternalAuthenticationInputValidParam - Otherwise` — branch entered when `OriginType` empty, restores SLO snapshot from DB
- OK — `genexus.security.api.RepositoryInputExternalAuthentication - ValidWhenIsIDP-True` — IDP-side return handler

Failure signature:
- FAIL — `genexus.security.api.GAMExternalAuthenticationInputValidParam - Otherwise - {"data":{"snapshot":null}}` — state TTL expired, snapshot not found, user redirected to login

### Cache — shared between SIGNIN and SLO (#cache)
See [GAM Cache Debugging](cache-debugging.md) for the full OAuth-flow cache-key catalog
- OK — `genexus.security.api.GAMGetCache - Cache-NotFound - {"data":{"CacheName":"com.genexus.gam.<id>","Key":"<key>"}}` — cache miss, benign if first access
- OK — `genexus.security.api.GAMGetCache - Cache-Found - {"data":{"CacheName":"com.genexus.gam.<id>","Key":"<key>"}}` — cache hit
- OK — `genexus.security.api.GAMSetCache - End_Method - {"data":{"CacheName":"com.genexus.gam.<id>","Key":"<key>","Value":{…}}}` — cache write

---

## How to use this catalog
Format A (`GAMTrace-` prefix, up to v18u14) and Format B (`genexus.security.api.*` structured JSON, v18u15+) must be read alongside each other. Both use the same 5-pass methodology from [GAM Trace Analyzer](common-trace-analyzer.md); only the signature shape differs

- Identify the domain from the question (auth, authz, slo, saml, otp, multitenant, impersonation)
- Jump to the relevant section
- Cross-reference present and absent traces against the flow definition in the domain file
- Apply 5-pass methodology from [GAM Trace Analyzer](common-trace-analyzer.md)

The absence of an expected trace is stronger evidence than the presence of an error
