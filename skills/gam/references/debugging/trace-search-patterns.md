---
name: trace-search-patterns
description: Per-domain expected trace sequences and error pattern catalog — used during Pass 2-4 of the 5-pass trace analysis methodology
---

# Trace Search Patterns and Error Catalog
Core methodology and format detection: [GAM Trace Analyzer](common-trace-analyzer.md). Load this file when doing a Pass 2-4 deep dive on a specific domain, or when checking a symptom against the Error Pattern Catalog

---

## Search patterns by domain
### Authentication (OAuth 2.0 / OIDC)
Read [GAM Authentication](../authentication/external-providers/common/domain-auth.md)

```
# Expected sequence for OAuth 2.0:
1. GAMTrace-Oauth20-Parameters:<authURL>
2. (state persisted across redirects, recoverable via the state parameter)
3. GAMTrace-ValidStateInDB OK State=<state>
4. GAMTrace-Oauth20-Token URL:<tokenEndpoint>
5. GAMTrace-Oauth20-Token Response:<json>
6. GAMTrace-GAMRemote AddHeader: Authorization:<tokenType> <accessToken>   <- CRITICAL
7. GAMTrace-Oauth20-User Response:<userInfoJson>

# If it stops at step 5 (Token Response OK but no AddHeader) -> Silent Skip
# If it stops at step 4 (Token URL but no Response) -> IDP rejected the request
# If it stops at step 3 (no ValidState) -> State expired or CSRF
```

v18u15+ — Expected sequence for OAuth 2.0 (IDP side, web flow):
```
1. genexus.security.api.GAMExternalAuthenticationInputValidParam - Start_Sub-ValidOriginTypeAndLoadParm - {"data":{"GAMExternalAuthenticatinInputSDT":"auth"}}
2. genexus.security.api.GAMExternalAuthenticationInputValidParam - Authentication-OAuth-20
3. genexus.security.api.GAMExternalAuthenticationInputValidParam - LoadParmToSDTExternalAuthenticationInput-scope - {"data":{"scope":"…"}}
4. genexus.security.api.GAMExternalAuthenticationInputValidParam - Start_Sub-ValidOriginTypeOAuth - {"data":{"Parm_ClientId":"…"}}
5. genexus.security.api.GAMApplicationValidScopes - End_Method - {"data":{"Scope":"…","ValidatedScopes":true,…}}
6. genexus.security.api.ValidApplicationOAuth20Protocol - End_Method - {"data":{"MustValidPKCE":true,"Errors":[]}}
7. genexus.security.api.GAMExternalAuthenticationInputValidParam - Start_Sub-ValidStateInDB - {"data":{"GAMTokenState":"GRESTD…"}}
8. genexus.security.api.GAMExternalAuthenticationGAMRemote - Start_SubOauth20SignIn
9. genexus.security.api.GAMExternalAuthenticationGAMRemote - End_Method - {"data":{"RedirToURL":"http://…idplogin.aspx?GRESTD…","Errors":[]}}

# If it stops at step 5 (ValidatedScopes:false) -> Scope not authorized
# If it stops at step 6 (Errors non-empty) -> OAuth protocol invalid (PKCE, etc.)
# If step 7 does not appear -> State expired, CSRF, or redirect state lost
# If step 9 shows Errors -> IDP rejected, callback URL invalid, etc.
```

v18u15+ — Expected sequence for local login (IDP page):
```
1. genexus.security.api.GAMAuthenticationLogin - AuthenticationType-Impersonate - {"data":{"isImpersonate":false,"AuthenticationTypeName":"local"}}
2. genexus.security.api.GAMAuthenticationLoginGAMLocal - Start_Sub-SecurityGAMLocal-Local
3. genexus.security.api.GAMAuthenticationLoginGAMLocal - SecurityGAMLocal-User-Local
4. genexus.security.api.GAMValidUserRepositoryAccess - End_Method - {"data":{"Errors":[]}}         <- user has repo access
5. genexus.security.api.GAMAuthenticationLoginGAMLocal - SecurityGAMLocal-User-and-Password-OK      <- credentials valid
6. genexus.security.api.GAMUserValidRequiredDataRepository - End_Method - {"data":{"CompleteUserDataOK":true}}
7. genexus.security.api.RepositoryLogin - Login-GAMSessionJSON_SDT: - {"data":{"GAMSessionJSON_SDT":{…}}}
8. genexus.security.api.GAMSessionAPI - Start_Sub-LoginUserInWebSession

# If step 4 shows Errors -> user lacks repository access
# If step 5 absent after step 3 -> incorrect password
# If step 6 shows CompleteUserDataOK:false -> required data missing in profile
```

### SLO (Single Log Out)
Read [GAM Single Log Out (SLO)](../logout/domain-slo.md) + [GAM Behavioral Patterns](./common-behavioral-patterns.md)

```
# Expected sequence for SLO:
1. GAMTrace-SLOProcess: Start
2. GAMTrace-SLOProcess Step1: TokenToFinish=<token>
3. GAMTrace-Logout_internal (first call - close child session)
4. GAMTrace-SLOProcess Step2: External IDP redirect
5. (redirect to external IDP to close session there)
6. GAMTrace-ValidStateInDB OK State=SLOInt…
7. GAMTrace-Logout_internal (second call - close parent session)
8. GAMTrace-SLOProcess: Complete, redirect to FromURL
```

v18u15+:
```
1. genexus.security.api.GAMDeleteWebSessionAndCookie - Start_Method - {"data":{"Parm1":…}}
2. genexus.security.api.GAMDeleteWebSessionAndCookie - Get-WebSessions - {"data":{"Session.UserGUID":"…","GAMConCli":{…}}}
3. genexus.security.api.GetScriptPathToCookie - End_Method - {"data":{"ScriptPath":"…"}}
4. genexus.security.api.GAMAntiFixationInit - Set-Cookie-and-WebSession - {"data":{"Value":"…"}}
# (For SLO with external IDP, also search: SLO, LogoutType, TokenToFinish in event labels)
```

### SAML 2.0
Read [GAM SAML 2.0 Reference](../authentication/external-providers/saml20/common/domain-saml.md)

```
# Expected sequence for SAML:
1. GAMTrace-SAML-Request:<base64AuthnRequest>
2. (redirect to SAML IDP)
3. GAMTrace-SAML-Response:<base64SAMLResponse>
4. GAMTrace-SAML: Assertion Issuer=<issuer>
5. GAMTrace-SAML: Signature validation OK
6. GAMTrace-SAML: Attributes extracted:<attributes>
```

v18u15+ (SAML still retains some legacy `GAMTrace-` format traces):
```
# The 3 active SAML traces in u15+ that use legacy format:
GAMTrace-<TraceTXT>-KeyStoreFilePathTrustCred: <path>
GAMTrace-<TraceTXT>-KeyStorePwdTrustCred: <pwd>
GAMTrace-<TraceTXT>-KeyAliasTrustCred: <alias>
# Rest of SAML traces use Format B (genexus.security.api.*)
```

### 2FA/OTP
Read [GAM OTP & 2FA Encyclopedia](../authentication/local-idp/domain-otp-2fa.md)

```
# Expected sequence for Login + 2FA:
1. GAMTrace-==+++GAMAuthenticationLogin ====== START ---
2. GAMTrace-GAMAuthenticationLogin 2FA - Valid &AuthenticationType2FA:<value>
3. (if value > 0: 2FA required)
4. GAMTrace-2FA: First factor OK, awaiting second factor
5. GAMTrace-OTP: Code sent to <email>  (if OTP)
6. GAMTrace-2FA: Second factor validated, session created
```

v18u15+:
```
1. genexus.security.api.GAMAuthenticationLogin - Start_Method - {"data":{…}}
2. genexus.security.api.GAMSessionAPI - Start_Sub-GenerateOTPSession - {"data":{…}}
# (Search event labels with OTP, 2FA, TwoFactor, SecondFactor)
```

---

## Error Pattern Catalog
- Token Response OK -> NO AddHeader -> Code 114 — Root cause: Silent Skip due to empty `token_type` — [Silent Skip Anti-Pattern](../authentication/external-providers/common/domain-auth.md#silent-skip-anti-pattern--complete-reference)
- ValidStateInDB -> NO match -> State error — Root cause: State expired or CSRF — [GAM Session and Token Reference](../authentication/external-providers/common/domain-session-token.md)
- Session expired but UI looks logged in — Root cause: Timeout conflict where OauthTokenExpire < WebSessionTimeout — [Error Codes](../authorization/domain-authz.md#error-codes)
- SAML Response received -> assertion FAILED — Root cause: Incorrect IDP certificate or EntityId mismatch — [GAM SAML 2.0 Reference](../authentication/external-providers/saml20/common/domain-saml.md)
- Login OK -> 2FA required -> timeout — Root cause: FirstAuthenticationFactorExpiration too low — [GAM OTP & 2FA Encyclopedia](../authentication/local-idp/domain-otp-2fa.md)
- SLOProcess starts -> hangs at external IDP — Root cause: IDP not responding to SLO redirect — [SLO External IDP Analysis Playbook](../logout/domain-slo-external-idp.md)
- IDP- prefix mismatch — Root cause: Known bug u13HF/u14 prefix removal — [Shared-DB State Collision — IDP- Prefix](./common-behavioral-patterns.md#shared-db-state-collision--idp--prefix)

### v18u15+ — Error Patterns (Structured JSON)
- `ValidatedScopes:false` in GAMApplicationValidScopes — Root cause: Scope not authorized for the app — [GAM Authentication](../authentication/external-providers/common/domain-auth.md)
- `ValidStateInDB` -> `LoadState:false` — Root cause: State expired, CSRF, or redirect state was cleaned — [GAM Session and Token Reference](../authentication/external-providers/common/domain-session-token.md)
- `SecurityGAMLocal-User-Local` without `User-and-Password-OK` — Root cause: Incorrect password — [GAM Authentication](../authentication/external-providers/common/domain-auth.md)
- `GAMValidUserRepositoryAccess` -> non-empty `Errors` — Root cause: User lacks repository access — [GAM Authorization — Permission Evaluation](../authorization/domain-authz.md)
- `CompleteUserDataOK:false` in GAMUserValidRequiredDataRepository — Root cause: Required data missing in profile — [GAM Authentication](../authentication/external-providers/common/domain-auth.md)
- `Start_Method` without corresponding `End_Method` — Root cause: Procedure aborted (exception, timeout, deadlock) — Cross-cutting
- `End_Method` with `"Errors":[{…}]` — Root cause: Procedure returned an explicit error — Depends on the procedure
- `GAMGetCache - Cache-NotFound` repeated — Root cause: Cache not persisting or expired — [GAM Connection & Configuration Encyclopedia](../multi-tenant/domain-connection-config.md)
- `ReadWebSession-WebSession-not-found` -> `Start_Sub-GiveAnonymousSession` — Root cause: No web session exists, anonymous session granted — [GAM Session and Token Reference](../authentication/external-providers/common/domain-session-token.md)
- `GAMDeleteWebSessionAndCookie` without subsequent SLO traces — Root cause: SLO chain interrupted, partial cleanup — [GAM Single Log Out (SLO)](../logout/domain-slo.md)
- SAML `GAMTrace-` lines coexist with Format B — Normal in u15+, SAML KeyStore traces were not migrated — [GAM SAML 2.0 Reference](../authentication/external-providers/saml20/common/domain-saml.md)
