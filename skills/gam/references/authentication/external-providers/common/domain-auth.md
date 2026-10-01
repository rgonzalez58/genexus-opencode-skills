---
name: domain-auth
description: Authentication flows — Local Login, OAuth 2.0, OIDC, SAML 2.0, Silent Skip Anti-Pattern. Deep-dives for IDP server endpoints, scopes, and error codes live in sibling files
---

# GAM Authentication
This reference covers how GAM authenticates users — the internal flows executed when a GeneXus app authenticates locally or against an external IDP. For the reverse direction (GAM acting as an IDP serving other clients), see `local-idp/authorization-code-flow.md` and `local-idp/password-grant-flow.md`

Related files:
- [GAM as Web IDP Server — Authorization Code Flow](../../local-idp/authorization-code-flow.md) — GAM as Web IDP (Authorization Code flow on `/oauth/gam`, PKCE, SLO handler)
- [GAM as REST IDP Server — Password Grant Flow](../../local-idp/password-grant-flow.md) — GAM as REST IDP (Password Grant flow on `/oauth/gam/v2.0`)
- [OAuth User Scopes](../../local-idp/scopes.md) — OAuth User Scopes (`gam_user_data`, `gam_user_roles`, custom scopes, identification rule)
- [Service-to-Service Authentication (`GAMAgentServiceHeader`)](service-to-service.md) — `GAMAgentServiceHeader` for propagating security context between backend services
- [GAM Authentication — Error Code Catalog](error-codes-auth.md) — Complete catalog of authentication error codes
- [GAM Authentication Configuration Reference](domain-auth-config.md) — Configuration templates for external authentication types
- [GAM SAML 2.0 Reference](../saml20/common/domain-saml.md) — SAML 2.0 deep dive
- [GAM OTP & 2FA Encyclopedia](../../local-idp/domain-otp-2fa.md) — 2FA / OTP flows
- [GeneXus Patterns for GAM Integration](../../../debugging/common-genexus-patterns.md) — GeneXus patterns when integrating with GAM External Objects

---

## Local Login (`GAMLogin` / `GAMLoginMobile`)
### Flow
```
User → Credentials (user/pass) → GAM.Login()
  → Look up user by username
  → Check: active? locked? password expired?
  → Validate password (hash comparison)
  → If 2FA enabled → Redirect to OTP verification
  → Create GAMSession → Return token
```

### Login Validation Sequence
GAM validates credentials in this order:

- Looks up user by username
- Checks if user is active → If not: Error Code 12 (UserNotActive)
- Checks if user is locked → If yes: Error Code 17 (UserLocked)
- Validates password hash. If mismatch → increments failed attempts counter. If counter >= MaxFailedAttempts (Security Policy) → locks user
- Checks 2FA requirement → If enabled: redirects to OTP verification (see `domain-otp-2fa.md`)
- Creates session and returns token

Important: GAM does NOT distinguish "user not found" from "wrong password" — both return Code 10 (InvalidCredentials). This is by security design

### Trace Signatures — Local Login
```
OK: GAMTrace-==+++GAMAuthenticationLogin ====== START ---
OK: GAMTrace-GAMAuthenticationLogin - &User:<username>
OK: GAMTrace-GAMAuthenticationLogin 2FA - Valid &AuthenticationType2FA:<value>
OK: GAMTrace-GAMAuthenticationLogin - Session created
FAIL: (absence of "Session created" = login failed — check error code in Errors)
```

Error codes for this flow live in `error-codes-auth.md`

### Social Login Shortcuts
GAM provides convenience methods for common OAuth providers:

```genexus
// Start Google login flow (redirects browser to Google)
GAMRepository.LoginGoogle()

// Start Twitter login flow
GAMRepository.LoginTwitter()

// Login against a remote GAM repository (GAMRemote)
GAMRepository.LoginGAMRemote()
```

These methods internally construct the OAuth Authorization URL for the corresponding pre-configured Authentication Type and redirect the browser. The Authentication Type must be configured in Backoffice with the provider's `client_id` and `client_secret`

---

## OAuth 2.0 (External IDP Authentication)
### Complete Flow
```
1. App → GAMRemote.Login() → Build Authorization URL
2. Redirect to external IDP: client_id, redirect_uri, scope, state, response_type=code
3. User authenticates at the external IDP
4. IDP redirects to GAM callback with: code, state
5. GAM → IDP Token Endpoint: exchange code → access_token + token_type
6. GAM → UserInfo Endpoint: GET with Bearer token → user data
7. GAM → Auto-provisioning of user (if not in GAM)
8. GAM → Create session → Redirect to FromURL
```

### Steps 1-2: Authorization URL and State Storage
GAM constructs the authorization URL with standard OAuth 2.0 parameters (`client_id`, `redirect_uri`, `scope`, `state`, `response_type=code`) and redirects the browser to the external IDP

State anti-replay mechanism:
- GAM generates a hash-based state value
- Saves the state along with the FromURL, ApplicationId, and RepositoryId in the `LoginTmp` table
- On callback, GAM validates the state against `LoginTmp` and deletes it (one-time use)
- If the state is not found (expired or already used), GAM rejects the callback as potential CSRF

Trace: `GAMTrace-Oauth20-Parameters:<authorizationURL>`

### Step 4: Callback — State Validation
When the IDP redirects back to GAM with `code` and `state`:

- GAM reads `code` and `state` from the callback query string
- Looks up the state in `LoginTmp` table
- If found → marks as valid and deletes the record (anti-replay)
- If not found → state invalid/expired, possible CSRF or cleaned by `GAMLoginTmpDemon`

Trace: `GAMTrace-ValidStateInDB OK State=<state>`

### Step 5: Token Exchange — CRITICAL
GAM exchanges the authorization code for tokens via a server-to-server POST to the IDP's token endpoint:

- Sends `grant_type=authorization_code`, `code`, `redirect_uri`, `client_id`, `client_secret` as form-encoded body
- On HTTP 200: parses the JSON response extracting `access_token`, `token_type`, `refresh_token`, `id_token`, `expires_in`
- On failure: logs the error and returns Code 114 (InvalidToken)

Expected fields in token response:

- `access_token` (required): Token for UserInfo request
- `token_type` (required): Must be non-empty (typically "Bearer") — empty value triggers Silent Skip
- `refresh_token` (optional): For session renewal
- `id_token` (OIDC only): JWT with identity claims
- `expires_in` (optional): Seconds until expiration

Traces:
- `GAMTrace-Oauth20-Token URL:<tokenEndpoint>`
- `GAMTrace-Oauth20-Token Response:<json>` (on success)
- `GAMTrace-Oauth20-Token FAILED Status:<statusCode>` (on failure)
- `GAMTrace-Oauth20-Token Error:<responseBody>` (on failure)

### Step 6: UserInfo with Bearer Token — Silent Skip Point
CRITICAL: This is the exact point where the Silent Skip Anti-Pattern occurs

GAM constructs the Authorization header from the token response and calls the IDP's UserInfo endpoint:

- If `token_type` is non-empty → builds `Authorization: <token_type> <access_token>` header
- If SSO REST mode → also adds `Client_id` header — see [SSO REST — Single Sign-On for REST Services](../../sso-rest/domain-sso-rest.md)
- Executes GET to the UserInfo endpoint

If `token_type` is empty, the entire header construction is silently skipped — no error, no trace. The UserInfo request goes without an Authorization header, the IDP rejects with 401, and GAM returns Code 114

DETECTION: ABSENCE of trace `GAMTrace-GAMRemote AddHeader: Authorization` after `Token Response` confirms Silent Skip

Trace (success): `GAMTrace-GAMRemote AddHeader: Authorization:<tokenType> <accessToken>`

### Step 7: Auto-provisioning
After obtaining user data from UserInfo:

- GAM searches for an existing user by external ID
- If user not found → creates a new user via Business Component with the data from UserInfo (name, email, external ID, active=true)
- If user found → proceeds with existing user
- Creates GAM session

Trace: `GAMTrace-Oauth20-User Response:<userInfoJson>`

### Trace Signatures — Complete OAuth 2.0
```
OK: GAMTrace-Oauth20-Parameters:<authorizationURL>
OK: (state saved to LoginTmp)
--- redirect to external IDP, user authenticates ---
OK: GAMTrace-ValidStateInDB OK State=<state>
OK: GAMTrace-Oauth20-Token URL:<tokenEndpoint>
OK: GAMTrace-Oauth20-Token Response:<json>
OK: GAMTrace-GAMRemote AddHeader: Authorization:<tokenType> <accessToken>   ← CRITICAL
OK: GAMTrace-Oauth20-User Response:<userInfoJson>
OK: (user provisioned/found, session created)
```

CRITICAL: If you see `Token Response` but NOT `AddHeader: Authorization`, this is the Silent Skip Anti-Pattern. The `token_type` in the IDP response is empty

### ResponseUserEmail_Name Mapping
GAM maps IDP claims to its internal fields. If the IDP changes claim names, GAM cannot find the data:

- Google: email field = `email`, name field = `name` — default in GAM
- Microsoft Entra: email field = `mail` or `userPrincipalName`, name field = `displayName` — adjust `ResponseUserEmail_Name` in Backoffice
- Facebook: email field = `email`, name field = `name` — default in GAM
- Keycloak: email field = `email`, name field = `preferred_username` — adjust in Backoffice
- Custom: configurable — Backoffice → Auth Type → User Info tab

Full provider-specific UserInfo mappings live in `external-providers/` (see `domain-auth-config.md`)

---

## OpenID Connect (OIDC)
### Differences from plain OAuth 2.0
- Identity token: OAuth 2.0 has none; OIDC provides `id_token` (JWT)
- UserInfo endpoint: OAuth 2.0 requires it; OIDC makes it optional (claims can be in JWT)
- Minimum scope: OAuth 2.0 is free-form; OIDC MUST include `openid`
- Validation: OAuth 2.0 checks only StatusCode; OIDC performs JWT signature + claims validation
- Discovery: OAuth 2.0 is manual; OIDC uses `.well-known/openid-configuration`

### JWT Validation Sequence
When GAM receives an `id_token` (JWT), it performs strict validation:

- Decode header → extract `kid` (Key ID)
- Fetch JWKS from the IDP's JWKS endpoint → find the public key matching `kid`
- Verify signature using the public key
- Validate claims strictly:
	* `iss` (issuer) must match the configured Issuer URL → if not: Code 515
	* `aud` (audience) must match the configured Client ID → if not: Code 515
	* `exp` (expiration) must be in the future → if not: Code 515

### Trace Signatures — OIDC
```
OK: GAMTrace-OIDC-&IDTokenHeader:<base64Header>
OK: GAMTrace-OIDC-&IDTokenPayload:<base64Payload>
OK: GAMTrace-OIDC-jwks_uri:<jwksUrl>
OK: (if JWT validation OK → continues as normal OAuth 2.0)
FAIL: GAMTrace-OIDC-Error when on JWT verification:<error detail>
```

For OIDC certificate download / rotation, see `oidc-certificate-management.md`

---

## SAML 2.0
For detailed configuration, attribute mapping, and certificates, see `domain-saml.md`

### SP-initiated Flow
```
1. Protected app → GAM detects no session
2. GAM builds AuthnRequest (XML)
3. Base64 encode → Redirect to SAML IDP
4. User authenticates at IDP
5. IDP sends SAML Response (HTTP POST) to GAM ACS
6. GAM parses and validates the SAML Assertion
7. GAM extracts user attributes → auto-provisioning
8. GAM creates session → Redirect to app
```

### Authentication Type Routing
When GAM receives an external authentication callback, it routes to the appropriate handler based on the configured authentication type:

- OAuth 2.0 → OAuth 2.0 token exchange flow
- SAML 2.0 → SAML assertion processing flow
- OTP → One-time password verification flow

### Trace Signatures — SAML
```
OK: GAMTrace-SAML-Request:<base64AuthnRequest>
OK: GAMTrace-SAML-Response:<base64SAMLResponse>
OK: (assertion parsing and validation)
FAIL: (if assertion validation fails — check error in response)
```

### Certificates and EntityID
For keystore formats per generator (`.pfx` for .NET, `.jks` for Java) and certificate diagnostics, see `domain-saml.md`

CRITICAL: `EntityID` or `AssertionConsumerServiceURL` mismatch causes immediate parsing failure

---

## Silent Skip Anti-Pattern — Complete Reference
### Definition
Occurs when a field in the IDP Token Response has an empty value. GAM uses an emptiness guard before acting on each field — if the field is empty, the dependent action is silently skipped. No error is thrown, no trace is emitted, no warning is generated

### Behavioral Description
When GAM receives the token response from the IDP, it checks each critical field before using it. If the field is empty, GAM skips the entire block that depends on that field without any error or trace output. This means:

- The token response arrives and is logged
- GAM checks if the field is non-empty
- If empty → the code block that uses that field is completely skipped
- Execution continues as if everything is normal
- The next step that depends on the skipped action fails (typically with a 401 from the IDP)

### Susceptible fields
- `token_type`: when empty, Authorization header is NOT constructed or sent → UserInfo request goes without Authorization → IDP returns 401 → Code 114
- `access_token`: when empty, cannot perform UserInfo request → Code 114
- `refresh_token`: when empty, refresh token is not stored → session cannot be renewed — must re-authenticate
- `id_token`: when empty, JWT validation is skipped → falls back to plain OAuth 2.0 flow (UserInfo only)
- `Client_id` (SSO REST): when empty, client identification header not sent → IDP cannot identify the requesting client — see [SSO REST — Single Sign-On for REST Services](../../sso-rest/domain-sso-rest.md)

### Detection checklist (6 steps)
```
1. OK: Find trace "Token Response" — is it present? Does it contain valid JSON?
2. OK: Parse the JSON from Token Response — does token_type have a value? Does access_token have a value?
3. OK: Find trace "AddHeader: Authorization" — is it PRESENT after the Token Response?
4. FAIL: If "AddHeader" is ABSENT → Silent Skip confirmed
5. OK: Identify WHICH field is empty by comparing the Token Response JSON
6. OK: Fix: configure the external IDP to include the missing field in the response
```

The pattern: "Received a response → a subsequent request failed → but there is no error in between" almost always means a Silent Skip

---

## grant_type Options
- `authorization_code`: standard OAuth 2.0 flow — used for login with redirect to IDP
- `refresh_token`: renew access_token — used when `OauthTokenExpire` expires
- `client_credentials`: server-to-server (no user) — used for MiniApp tokens, API-to-API
- `password`: Resource Owner Password Credentials — used by REST IDP (see `local-idp/password-grant-flow.md`)
