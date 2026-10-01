---
name: provider-generic-oauth20
description: Programmatic initialization of GAMAuthenticationTypeOAuth20 — template-based pattern (preferred) and full property-by-property reference for any OAuth 2.0 IDP
---

# OAuth 2.0 Auth Type — Programmatic Initialization
Scope: how to create or update a `GAMAuthenticationTypeOAuth20` entity by code. Covers two patterns: template-based (preferred, uses built-in GAM defaults) and property-by-property (for full control or providers without a built-in template)

Related files:
- [GAM Authentication Configuration Reference](../common/domain-auth-config.md) — auth type configuration via Backoffice
- [Provider — Okta (OAuth 2.0 + OIDC)](provider-okta.md) — Okta endpoint reference
- [Provider — Keycloak (OAuth 2.0 + OIDC)](provider-keycloak.md) — Keycloak endpoint reference
- [Provider — Auth0 (OAuth 2.0 + OIDC)](provider-auth0.md) — Auth0 endpoint reference
- [Provider — Facebook](provider-facebook.md) — Facebook reference (dedicated EO)
- [OIDC Certificate Management](./common/oidc-certificate-management.md) — OIDC cert refresh

Variable types:
- `&GAMAuthenticationTypeOAuth20` — `GAMAuthenticationTypeOAuth20, GeneXusSecurity` (EO)
- `&GAMAuthenticationOAuth20` — `GAMAuthenticationOAuth20, GeneXusSecurity` (SDT — holds OAuth20 defaults from template)
- `&GAMErrorCollection` — `GAMError, GeneXusSecurity` — Collection: True
- `&GAMError` — `GAMError, GeneXusSecurity`

External Object schema reference (for property-by-property derivation):
- EO name: `GAMAuthenticationTypeOAuth20`, module `GeneXusSecurity`
- To inspect the full property and method list, delegate to Nexa: ask it to look up the External Object `GAMAuthenticationTypeOAuth20` in the `GeneXusSecurity` module of the active KB. Nexa will use `gx` to read its Members and derive the complete settable property tree

---

## Pattern 1 — Template-based initialization (preferred)
Uses built-in GAM defaults (`GAMInitAuthenticationTypeOAuth20` enum). Only override what's specific to your app

### Available templates
- `GAMInitAuthenticationTypeOAuth20.Microsoft` — Microsoft Entra ID (Azure AD) v1 — pre-fills: All MS endpoints, Graph UserInfo, SLO
- `GAMInitAuthenticationTypeOAuth20.Google` — Google — pre-fills: All Google endpoints, OIDC
- `GAMInitAuthenticationTypeOAuth20.Apple` — Sign in with Apple — pre-fills: Apple endpoints, PKCE
- `GAMInitAuthenticationTypeOAuth20.Twitter` — Twitter/X — pre-fills: Twitter OAuth 2.0 endpoints
- `GAMInitAuthenticationTypeOAuth20.Default` — Generic / custom IDP — pre-fills: Minimal structure, all fields blank

**Google, Twitter, and Apple also have dedicated built-in EOs** (`GAMAuthenticationTypeGoogle`, `GAMAuthenticationTypeTwitter`, `GAMAuthenticationTypeApple`, `GAMAuthenticationTypeFacebook`) where GAM handles the OAuth flow internally. Use those when you only need to configure ClientId, ClientSecret, and SiteURL. Use these templates only when you need full control over endpoints, scopes, UserInfo mapping, or SLO — i.e., when you explicitly want `GAMAuthenticationTypeOAuth20`

**These two approaches cannot be combined.** A single auth type configuration uses either `GAMAuthenticationTypeOAuth20` (with or without a template) or the built-in EO — never both. See the "Built-in provider EOs" section below

**Internal behavior of template Load:** Calls `AuthenticationTypeOAuth20API_internal` with `APIMode=Display`. The name matches `GAMInitAuthenticationTypeOAuth20.Elements()` → fills the OAuth20 SDT with provider defaults → resets `Name=""`. No DB query. Returns `Success()=True`

### Usage
Idempotent Load-or-New + Save/error shape — see [Idempotent Save Pattern](../../../debugging/common-genexus-patterns.md#idempotent-save-pattern-load-or-new), with one twist: `Load(GAMInitAuthenticationTypeOAuth20.<Provider>)` (not `Load(&Name)`) loads the template into `&GAMAuthenticationOAuth20`, then `= new()` resets Insert mode, then re-assign `Name`/`IsEnable`/`OAuth20 = &GAMAuthenticationOAuth20` (the template SDT) before overriding `ClientId_Value`/`ClientSecret_Value`/`RedirectURL_Value` with your app's values

**Why `= new()` is required after loading a template:** `Load(template)` leaves the EO in "loaded" state — a `Save()` from there would attempt an Update with no matching DB record (Error 42). `= new()` resets to Insert mode; the template SDT captured before the reset supplies the provider defaults

Look up exact property names/types on the `GAMAuthenticationTypeOAuth20` EO (EO Verification Protocol) — override only `ClientId_Value`, `ClientSecret_Value`, `RedirectURL_Value` for most templates

---

## Pattern 2 — Property-by-property initialization (full control)
Use this when:
- The provider has no built-in template (Okta, Keycloak, Auth0, Facebook, LinkedIn)
- You need to set every property explicitly
- The skill needs to derive the full property list from the External Object

Idempotent Load-or-New + Save/error shape — see [Idempotent Save Pattern](../../../debugging/common-genexus-patterns.md#idempotent-save-pattern-load-or-new). For the complete property tree (General, Authorize + PKCE + OIDC, Token, UserInfo, Roles, Signout — every property with type, default, and purpose), see [OAuth 2.0 Property Tree (AuthenticationOAuth20SDT)](../common/oauth20-properties.md); confirm exact names against the EO itself (EO Verification Protocol)

Minimum properties needed for any provider: top-level `Name`, `IsEnable`; `OAuth20.ClientId_Value` / `ClientSecret_Value` / `RedirectURL_Value`; `Authorize.URL`; `Token.URL`; `UserInfo.URL`. Enable `Authorize.PKCEAuthentication.Enable` for PKCE-requiring IDPs (e.g., Twitter/X) and `Authorize.OpenIDConnectAuthentication.Enable` for OIDC providers. Enable `Signout.SLOEnable` + `Signout.URL` if the IDP supports SLO

---

## Built-in provider EOs
Google, Twitter, and Facebook have dedicated GAM External Objects where **GAM handles the entire OAuth flow internally** — the developer only sets `ClientId`, `ClientSecret`, `SiteURL` (plus `Name`/`IsEnable`/`Description`/`Impersonate` at the top level). Same idempotent shape as Pattern 1/2

- Google → `GAMAuthenticationTypeGoogle`, sub-SDT `.Google.*`
- Twitter/X → `GAMAuthenticationTypeTwitter`, sub-SDT `.Twitter.*`
- Apple → `GAMAuthenticationTypeApple`, sub-SDT `.Apple.*`
- Facebook → `GAMAuthenticationTypeFacebook`, sub-SDT `.Facebook.*` — see [Provider — Facebook](provider-facebook.md)

All in module `GeneXusSecurity`. **Cannot mix with `GAMAuthenticationTypeOAuth20`:** these are separate entity types — a single auth type record is initialized with either the built-in EO or `GAMAuthenticationTypeOAuth20`, never both

---

## Provider-specific files
The individual `provider-*.md` files in this folder contain **only what is specific or different** for that IDP. Use them for endpoint URLs and IDP-specific quirks

Providers with both a built-in EO and an OAuth20 template (choose one approach, cannot combine):
- Google → built-in: `GAMAuthenticationTypeGoogle` / OAuth20 template: `.Google`
- Twitter → built-in: `GAMAuthenticationTypeTwitter` / OAuth20 template: `.Twitter`
- Apple → built-in: `GAMAuthenticationTypeApple` / OAuth20 template: `.Apple`

Provider with built-in EO only (no OAuth20 template in the enum):
- Facebook → `GAMAuthenticationTypeFacebook` (see `provider-facebook.md` for Option B using `GAMAuthenticationTypeOAuth20` Pattern 2)

Provider with OAuth20 template only (no built-in EO):
- Microsoft → `.Microsoft`

Providers with no built-in template — use Pattern 2 with endpoints from their specific file:
- Okta → see `provider-okta.md` — custom auth server URL pattern, revoke-based signout
- Keycloak → see `provider-keycloak.md` — realm-based URLs, `id_token_hint`, roles via token claim
- Auth0 → see `provider-auth0.md` — `audience` parameter, `returnTo` logout
- LinkedIn → no dedicated file; use Pattern 2 with `GAMInitAuthenticationTypeOAuth20.Default`

---

## Internal mechanics (why Load(template) works the way it does)
`GAMInitAuthenticationTypeOAuth20.Microsoft` returns the string `"GAMInitAuthTypeOAuth20-Microsoft"`. When passed to `Load()`, `AuthenticationTypeOAuth20API_internal` detects it as a template key via `GAMInitAuthenticationTypeOAuth20.Elements()`, calls `GAMInitAuthenticationTypeOAuth20_Microsoft(...)` to fill the OAuth20 SDT, then resets `Name=""`. The call to `GAMAuthenticationTypeAPI` is **skipped entirely** — no DB query. `Success()` returns `True` with `Name=""` and `.OAuth20` filled with Microsoft defaults

**Validation rules enforced on Save() (Insert/Update mode only):**
- `ClientId_Value` must not be empty (Error: `AuthenticationTypeClientIdCannotBeNull`)
- `RedirectURL_Value` must not be empty (Error: `AuthenticationTypeSiteURLCannotBeNull`)
- If OIDC is enabled + ValidIDToken: `IssuerURL` and (`CertificatePathFileName` or `DiscoveryURL`) must not be empty
- If PKCE is enabled: `LenghtChallenge` must be ≥ 32
- If OIDC is enabled: `Scope_Include` must be True
- If `FunctionId = AuthenticationAndRoles`: `Roles.URL` must not be empty
- If `Signout.SLOEnable = True`: `Signout.URL` must not be empty
