---
name: domain-auth-config
description: Hub for external authentication configuration — auth types, provider comparison, Backoffice mapping, troubleshooting. Property trees live in oauth20-properties.md / saml20-config.md
---

# GAM Authentication Configuration Reference
## DEFINITION
This reference is the hub for all GAM external authentication configuration. It covers:

- All GAM authentication types and their common properties
- A side-by-side provider comparison
- How API properties map to GAM Backoffice UI fields
- Troubleshooting and Decision Points for the whole authentication configuration surface

The complete property trees live in their own files — [OAuth 2.0 Property Tree (AuthenticationOAuth20SDT)](oauth20-properties.md) (`AuthenticationOAuth20SDT`) and [SAML 2.0 Configuration (AuthenticationTypeSAML20SDT)](saml20-config.md) (`AuthenticationSAML20SDT`) — load them only when configuring that type property-by-property

Dedicated configuration templates exist for Okta, Keycloak, Auth0, and Facebook under `external-providers/oauth20/`. Microsoft, Google, Apple, Twitter, and LinkedIn have no dedicated file — use the generic starter (`../oauth20/provider-generic-oauth20.md`) with the endpoint values from Provider Comparison below

Related files:
- [OAuth 2.0 Property Tree (AuthenticationOAuth20SDT)](oauth20-properties.md) — complete OAuth 2.0 property tree
- [SAML 2.0 Configuration (AuthenticationTypeSAML20SDT)](saml20-config.md) — complete SAML 2.0 property tree
- [Provider — Okta (OAuth 2.0 + OIDC)](../oauth20/provider-okta.md) — Okta
- [Provider — Keycloak (OAuth 2.0 + OIDC)](../oauth20/provider-keycloak.md) — Keycloak
- [Provider — Auth0 (OAuth 2.0 + OIDC)](../oauth20/provider-auth0.md) — Auth0
- [Provider — Facebook](../oauth20/provider-facebook.md) — Facebook (dedicated type + full OAuth 2.0)
- [OAuth 2.0 Auth Type — Programmatic Initialization](../oauth20/provider-generic-oauth20.md) — Starter template for any custom IDP; also covers Microsoft, Google, Apple, Twitter, and LinkedIn (no dedicated file — see Provider Comparison below)
- [OIDC Certificate Management](../oauth20/common/oidc-certificate-management.md) — Download / rotate OIDC signing certificates
- [GAM SAML 2.0 Reference](../saml20/common/domain-saml.md) — SAML 2.0 deep dive (SP/IDP roles, metadata, attribute mapping)
- [GAM Entity Initialization — Consolidated Pattern](../../../kb-setup/init/entity-initialization-consolidated.md) — Base initialization pattern

---

## GAM Authentication Types
GAM supports the following types, each with its own External Object:

- `Local` (built-in): Username/Password against GAM DB
- `Apple` (`GAMAuthenticationTypeApple`): Sign in with Apple
- `Facebook` (`GAMAuthenticationTypeFacebook`): Facebook Login
- `Google` (`GAMAuthenticationTypeGoogle`): Google Sign-In
- `Twitter` (`GAMAuthenticationTypeTwitter`): Twitter/X OAuth
- `Instagram` (`GAMAuthenticationTypeInstagram`): Instagram Basic
- `LinkedIn` (`GAMAuthenticationTypeLinkedin`): LinkedIn OAuth 2.0
- `OAuth 2.0` (`GAMAuthenticationTypeOAuth20`): Generic (Azure, Okta, Keycloak, AGESIC, any provider)
- `SAML 2.0` (`GAMAuthenticationTypeSAML20`): SAML SP Federation
- `OTP`: One-Time Password second factor (email/SMS). See `domain-otp-2fa.md`. Wired as second factor via another AuthType's `TwoFactorAuthentication` sub-SDT
- `TOTP`: Time-based OTP second factor (authenticator app, QR enrollment). See `domain-otp-2fa.md`. Wired as second factor via another AuthType's `TwoFactorAuthentication` sub-SDT
- `Custom` (`GAMAuthenticationTypeCustom`): Custom authentication

---

## Common Properties for All Types
Every auth type EO shares these top-level properties: `Name` (unique), `IsEnable`, `Description`, `Impersonate`, `SmallImageName`, `BigImageName`

### Idempotent Pattern for Auth Types
Same Load-or-New + Save/error shape as every other GAM entity — see [Idempotent Save Pattern](../../../debugging/common-genexus-patterns.md#idempotent-save-pattern-load-or-new) for the canonical pattern. Applied to an auth type: `&GAMAuthType.Load(&Name)` → `If not Success() → &GAMAuthType = new()` → set properties → `Save()` → `GetErrors()`/`Commit` on failure/success

### Delete All Auth Types (reset)
Iterate `GAMRepository.GetAuthenticationTypes(&filter, &errors)` and call the static `GAMAuthenticationType.Delete(<name>, &errors)` for each — commit inside the loop after each successful delete

---

## OAuth 2.0 Property Tree
Full property tree (General, Authorize + PKCE + OIDC, Token, UserInfo, Roles, Signout — every property with type, default, purpose): [OAuth 2.0 Property Tree (AuthenticationOAuth20SDT)](oauth20-properties.md)

---

## Provider-Specific Templates
Every supported provider has a dedicated file under `external-providers/` with a ready-to-paste GeneXus template, the exact endpoint URLs, UserInfo field mapping, and any provider-specific quirks

### Provider Comparison
- Microsoft (v2)
	* Protocol: OAuth 2.0 + OIDC
	* OIDC: Optional but recommended
	* PKCE: Optional
	* Authorize URL: `login.microsoftonline.com/<tenant>/authorize`
	* Token URL: `login.microsoftonline.com/<tenant>/token`
	* UserInfo URL: `graph.microsoft.com/v1.0/me`
	* UserInfo Method: GET (Bearer header)
	* Scope: `openid email profile`
	* Token Auth: Body params
	* Email available: Yes (`mail`)
	* SLO supported: Yes (redirect)
	* ExternalId field: `id`
	* Roles endpoint: Yes (Graph API)
	* Additional params: None (v2) / `resource=` (v1)
	* Template: `../oauth20/provider-generic-oauth20.md` (`GAMInitAuthenticationTypeOAuth20.Microsoft`)
- Google
	* Protocol: OAuth 2.0 + OIDC
	* OIDC: Optional
	* PKCE: Not required
	* Authorize URL: `accounts.google.com/o/oauth2/auth`
	* Token URL: `accounts.google.com/o/oauth2/token`
	* UserInfo URL: `www.googleapis.com/oauth2/v1/userinfo`
	* UserInfo Method: GET (query param token)
	* Scope: `openid email profile`
	* Token Auth: Body params
	* Email available: Yes (`email`)
	* SLO supported: Yes (revoke endpoint)
	* ExternalId field: `id`
	* Roles endpoint: No
	* Template: `../oauth20/provider-generic-oauth20.md` (`GAMInitAuthenticationTypeOAuth20.Google`)
- Apple
	* Protocol: OAuth 2.0 + OIDC (required)
	* OIDC: REQUIRED
	* PKCE: Not required
	* Authorize URL: `appleid.apple.com/auth/authorize`
	* Token URL: `appleid.apple.com/auth/token`
	* UserInfo URL: None (uses ID token)
	* Scope: `email name`
	* Token Auth: Body params
	* Email available: Yes (via ID token)
	* SLO supported: Yes (revoke endpoint)
	* ExternalId field: From ID token
	* Roles endpoint: No
	* Additional params: `response_mode=form_post`
	* Template: `../oauth20/provider-generic-oauth20.md` (`GAMInitAuthenticationTypeOAuth20.Apple`)
- Twitter/X
	* Protocol: OAuth 2.0 + PKCE (required)
	* OIDC: Not supported
	* PKCE: REQUIRED
	* Authorize URL: `twitter.com/i/oauth2/authorize`
	* Token URL: `api.twitter.com/2/oauth2/token`
	* UserInfo URL: `api.twitter.com/2/users/me`
	* UserInfo Method: GET (Bearer header)
	* Scope: `users.read tweet.read offline.access`
	* Token Auth: Basic Auth header
	* Email available: No
	* SLO supported: Yes (revoke endpoint)
	* ExternalId field: `data.id`
	* Roles endpoint: No
	* Template: `../oauth20/provider-generic-oauth20.md` (`GAMInitAuthenticationTypeOAuth20.Twitter`)
- Okta
	* Protocol: OAuth 2.0 + OIDC
	* Authorize URL: `<domain>.okta.com/oauth2/v1/authorize`
	* Discovery: `<domain>.okta.com/.well-known/openid-configuration`
	* Template: `../oauth20/provider-okta.md`
- Keycloak
	* Protocol: OAuth 2.0 + OIDC
	* Authorize URL: `<host>/realms/<realm>/protocol/openid-connect/auth`
	* Discovery: `<host>/realms/<realm>/.well-known/openid-configuration`
	* Template: `../oauth20/provider-keycloak.md`
- Auth0
	* Protocol: OAuth 2.0 + OIDC
	* Authorize URL: `<tenant>/authorize`
	* Discovery: `<tenant>/.well-known/openid-configuration`
	* Template: `../oauth20/provider-auth0.md`
- Facebook
	* Protocol: OAuth 2.0
	* Dedicated type available (`GAMAuthenticationTypeFacebook`) for simplified setup
	* Template: `../oauth20/provider-facebook.md`
- LinkedIn
	* Protocol: OAuth 2.0 + OIDC
	* Discovery: `www.linkedin.com/oauth/.well-known/openid-configuration`
	* Template: `../oauth20/provider-generic-oauth20.md` (`GAMInitAuthenticationTypeOAuth20.Default` — no dedicated template)
- Azure B2C (use generic OAuth 2.0)
	* Authorize: `<tenant>.b2clogin.com/<tenant>.onmicrosoft.com/<policy>/oauth2/v2.0/authorize`
	* Token: `<tenant>.b2clogin.com/<tenant>.onmicrosoft.com/<policy>/oauth2/v2.0/token`
	* Scope: `<client-id> offline_access`
	* Grant Type (password flow): `password` (set `RedirectToAuthenticate = False`)
	* Template: `../oauth20/provider-generic-oauth20.md`
- Any other IDP (AGESIC, custom, corporate SSO)
	* Template: `../oauth20/provider-generic-oauth20.md`

---

## SAML 2.0 Configuration (AuthenticationTypeSAML20SDT)
Full property tree + initialization pattern: [SAML 2.0 Configuration (AuthenticationTypeSAML20SDT)](saml20-config.md). For in-depth SAML documentation (SP/IDP roles, metadata, attribute mapping, signing), see `domain-saml.md`. For the Microsoft Entra ID SAML variant, see `../saml20/common/domain-saml.md` (Supported IDPs → Microsoft Entra ID)

---

## Dedicated Provider Types (Simplified API)
Three providers expose a simplified External Object that hides the underlying OAuth 2.0 plumbing. Use these when the default flow is sufficient; switch to the generic OAuth 2.0 type when you need full control over scope, UserInfo fields, or SLO

### Apple Sign-In (Dedicated Type)
Variable: `exo:GAMAuthenticationTypeApple, GeneXusSecurity`. Properties: top-level `Name`/`IsEnable`/`Description`/`Impersonate`; `Apple` sub-SDT `ClientId` (Apple Services ID), `ClientSecret` (JWT signed with Apple private key), `SiteURL` (MUST be HTTPS), `AdditionalScope`

For full control (OIDC settings, form_post response, custom UserInfo mapping), use the generic OAuth 2.0 type with the template in `../oauth20/provider-generic-oauth20.md` (`GAMInitAuthenticationTypeOAuth20.Apple`)

### Facebook (Dedicated Type)
Variable: `exo:GAMAuthenticationTypeFacebook, GeneXusSecurity`. See `../oauth20/provider-facebook.md` for the simplified and full OAuth 2.0 variants

### Google (Dedicated Type)
Variable: `exo:GAMAuthenticationTypeGoogle, GeneXusSecurity`. Similar to Facebook. ClientId and ClientSecret from Google Cloud Console. For full control, use the generic OAuth 2.0 type with the template in `../oauth20/provider-generic-oauth20.md` (`GAMInitAuthenticationTypeOAuth20.Google`)

---

## Custom Authentication
Variable: `exo:GAMAuthenticationTypeCustom, GeneXusSecurity`

Allows defining a custom GeneXus procedure to validate credentials. Properties: top-level `Name`/`IsEnable`/`Description`/`Impersonate` plus the custom-procedure reference — confirm exact property name against the EO (EO Verification Protocol)

---

## API-to-Backoffice Field Mapping
### Where to configure in the Backoffice
Path: GAM Backoffice → Settings → Authentication Types → Add/Edit

- `.Name` maps to Name
- `.IsEnable` maps to Enabled (checkbox)
- `.Description` maps to Description
- `.Impersonate` maps to Impersonate Authentication Type
- `.SmallImageName` maps to Small Image
- `.OAuth20.ClientId_Value` maps to Client ID
- `.OAuth20.ClientSecret_Value` maps to Client Secret
- `.OAuth20.RedirectURL_Value` maps to Redirect URL
- `.OAuth20.RedirectURL_isCustom` maps to Custom Redirect URL (checkbox)
- `.OAuth20.Authorize.URL` maps to Authorization URL
- `.OAuth20.Authorize.Scope_Value` maps to Scope
- `.OAuth20.Token.URL` maps to Token URL
- `.OAuth20.UserInfo.URL` maps to User Info URL
- `.OAuth20.Authorize.OpenIDConnectAuthentication.Enable` maps to OIDC Enabled (checkbox)
- `.OAuth20.Authorize.OpenIDConnectAuthentication.DiscoveryURL` maps to OIDC Discovery URL
- `.OAuth20.Authorize.PKCEAuthentication.Enable` maps to PKCE Enabled (checkbox)
- `.OAuth20.Signout.SLOEnable` maps to SLO Enabled (checkbox)
- `.OAuth20.Signout.URL` maps to Signout URL

### Backoffice Sections for Auth Types
```
Settings
	+-- Authentication Types
		+-- [List] — Lists all configured types
		+-- [Add/Edit]
			+-- General (Name, Description, Enable, Impersonate)
			+-- Client Credentials (ClientId, ClientSecret, RedirectURL)
			+-- Authorization (URL, ResponseType, Scope, State, Additional)
			+-- Token (URL, Method, Headers, GrantType, Response mapping)
			+-- User Info (URL, Method, Response field mapping)
			+-- OpenID Connect (Enable, IssuerURL, DiscoveryURL, ValidateIDToken)
			+-- PKCE (Enable, Method, ChallengeLength)
			+-- Roles (URL, Method, Response mapping)
			+-- Signout (SLOEnable, URL, Method, Redirect)
```

---

## Troubleshooting
### Common Errors When Configuring Auth Types
- Code 245 (redirect URL error)
	* Cause: `RedirectURL_Value` is empty or missing
	* Solution: Set `RedirectURL_Value` to your app URL
- "Authentication type name already exists"
	* Cause: Duplicate name
	* Solution: Use `Load(&Name)` before creating
- "Invalid redirect URL"
	* Cause: URL not registered in IDP
	* Solution: Register exact URL in IDP console
- Silent Skip (no visible error)
	* Cause: `token_type` empty in response
	* Solution: Verify IDP returns `token_type`
- OIDC validation fails
	* Cause: Expired certificate
	* Solution: Update OIDC certificate — see `oidc-certificate-management.md`
- "error_description" empty
	* Cause: Wrong error field name
	* Solution: Check `ResponseErrorDescription_Name` (some use `!"error"`, Twitter uses `!"errors"`)
- Apple login fails
	* Cause: Missing `response_mode=form_post`
	* Solution: Add to `Authorize.AdditionalParameters` — see `../oauth20/provider-generic-oauth20.md`
- Twitter token exchange fails
	* Cause: Missing Basic Auth
	* Solution: Set `Header_Authentication_Include = True` and `Header_AuthorizationBasic_Include = True` — see `../oauth20/provider-generic-oauth20.md`
- Twitter no email
	* Cause: Twitter API does not return email
	* Solution: Normal behavior — email field stays empty
- SAML EntityId mismatch
	* Cause: SP Entity ID does not match IDP config
	* Solution: Ensure `ServiceProviderEntityId` matches exactly what is registered in the IDP

### Verification Post-Configuration
- By code: `GAMRepository.GetAuthenticationTypes(&filter, &errors)` — verify count
- By backoffice: Settings → Authentication Types — verify listing
- Test: Attempt login with each configured type
- Traces: Enable GAM tracing (`../debugging/common-debugging.md`) to see full redirect chain and token exchange

---

## Decision Points
Before configuring authentication types, Claude MUST check which Decision Points apply and present them. If the user says "defaults", use DEFAULT values

### DP-1: Which Authentication Types to Configure?
- Trigger: User asks to configure authentication, add external login, or set up SSO
- Question: "Which authentication types do you need to configure?"
- Options:
	* `local` — Local authentication only (username/password against GAM DB). Enabled by default
	* `oauth_provider` — A specific OAuth 2.0 provider (Google, Facebook, Apple, Twitter, LinkedIn, etc.). Specify which one
	* `oidc` — Generic OpenID Connect (Azure/EntraID, Okta, Keycloak, etc.). Specify provider
	* `saml` — SAML 2.0 (requires additional `domain-saml.md`)
	* `multiple` — Multiple types. List which ones
- Impact: Determines which provider file under `external-providers/` to open
- Phase: configure

### DP-2: Configuration Method
- Trigger: For any auth type configuration
- Question: "Configure via GeneXus code (initialization procedure), via GAM Backoffice, or both (code + Backoffice verification)?"
- Options:
	* `code` — (DEFAULT) GeneXus procedure with idempotent pattern. Reproducible, versionable, automatable
	* `backoffice` — Configure manually in GAM Backoffice → Settings → Authentication Types. More visual, but not reproducible
	* `both` — Code to create + verification instructions in Backoffice
- Impact:
	* `code`: Full template is generated using the provider file in `external-providers/`. Refer to `../../../kb-setup/init/entity-initialization-consolidated.md` for the base initialization pattern
	* `backoffice`: Step-by-step instructions with field mapping (refer to `../backoffice/auth-types.md`)
	* `both`: Code + verification checklist in Backoffice
- Phase: configure

### DP-3: IDP Credentials — Does User Have Them?
- Trigger: When configuring any external auth type (OAuth, OIDC, SAML)
- Question: "Do you already have the provider credentials (Client ID, Client Secret, endpoints)? Or do you need guidance to obtain them?"
- Options:
	* `have_them` — User has Client ID, Secret, and endpoints. Proceed with configuration
	* `need_guidance` — Provide instructions to register the app with the provider (Google Console, Azure Portal, etc.) and obtain credentials
- Impact:
	* `have_them`: Go directly to the GAM configuration template in the matching `external-providers/` file
	* `need_guidance`: Add a preliminary step with IDP-side instructions (outside GAM scope, but useful for the user)
- Phase: configure

### DP-4: Redirect URL Base
- Trigger: When configuring OAuth 2.0 or OIDC (require callback URLs)
- Question: "What is your application's base URL? (example: `http://localhost/MyAppNetCoreSQL` or `https://mydomain.com/app`)"
- Options: (free text — user provides the URL)
- Impact: Used to calculate `RedirectURL_Value` = `<BaseURL>/oauth/gam/callback`. This value MUST match exactly what is registered in the IDP. Error 245 if it does not match
- Phase: configure
