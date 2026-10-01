---
name: domain-gamremote
description: GAM as SP — programmatic initialization of GAMAuthenticationTypeGAMRemote (Web Authorization Code flow against external GAM IDP) including Encrypted variant
---

# GAM as SP — GAMRemote (Web Flow) Initialization
Initializes `GAMAuthenticationTypeGAMRemote` by code — configures a GAM SP to authenticate users against an external GAM IDP via Authorization Code flow with PKCE (or without PKCE)

Related files:
- [GAM as Web IDP Server — Authorization Code Flow](../local-idp/authorization-code-flow.md) — runtime flow (IDP side)
- [GAM as SP — GAMRemoteRest (REST Password Grant) Initialization](./domain-gamremoterest.md) — REST sibling (Password Grant, no browser, 2FA support)
- [GAM Authentication Configuration Reference](../external-providers/common/domain-auth-config.md) — Backoffice auth-type configuration
- [GeneXus Patterns for GAM Integration](../../debugging/common-genexus-patterns.md) — error handling patterns
- [SSO REST — Single Sign-On for REST Services](../sso-rest/domain-sso-rest.md) — SSO REST token emission: when the IDP Application has `SSORESTEnable=True, SSORESTMode=Server`, this flow causes the IDP to include an SSO REST token in the response

Variable types:
- `&GAMAuthenticationTypeGAMRemote` — `GAMAuthenticationTypeGAMRemote, GeneXusSecurity` (EO)
- `&GAMErrorCollection` — `GAMError, GeneXusSecurity` — Collection: True
- `&GAMError` — `GAMError, GeneXusSecurity`

---

## Decision Points
### DP-1: Variant
- Trigger: User requests GAMRemote initialization
- Question: "Standard or Encrypted?"
- Options:
	* `standard` (DEFAULT) — Web Authorization Code flow, no extra key
	* `encrypted` — adds `RemoteServerKey`; IDP Application must have the matching key configured
- Phase: design

### DP-2: Credentials
- Trigger: Always
- Question: "Are `ClientId` and `ClientSecret` already known, or should they be generated?"
- Options:
	* `random` (DEFAULT) — generate deterministic values via `GAMHelper.GenerateSHA512`; see Random credentials helper
	* `provided` — user supplies literal values; collect before generating code
- Phase: design

### DP-3: RemoteServerURL
- Trigger: Always
- Question: "What is the base URL of the external GAM IDP?"
- Options: free text — format `https://<idp-server>/<idp-virtual-dir>/`
- Impact: REQUIRED. Use dummy `https://<your-idp-server>/<your-idp-virtual-dir>/` and warn user to replace if not provided
- Phase: build

### DP-4: SiteURL
- Trigger: Always
- Question: "What is the base URL of this application (the SP)?"
- Options: free text — format `https://<app-server>/<app-virtual-dir>/`
- Impact: REQUIRED — used as `redirect_uri` basis. Use dummy `https://<your-app-server>/<your-app-virtual-dir>/` and warn user to replace if not provided
- Phase: build

### DP-5: Auth type name
- Trigger: Always
- Question: "What name should this auth type have?"
- Options: free text
- DEFAULT: `gamremote` / `gamremote-encrypted` for DP-1 `encrypted`
- Phase: design

### DP-6: PKCE
- Trigger: Always
- Question: "Enable PKCE?"
- Options:
	* `enabled` (DEFAULT) — method `S256`, `LenghtChallenge = 40`
	* `disabled` — only if the IDP does not support PKCE
- Phase: design

### DP-7: SLO
- Trigger: Always
- Question: "Enable SLO (logout propagation to the IDP)?"
- Options:
	* `enabled` (DEFAULT)
	* `disabled`
- Phase: design

### DP-8: Scopes
- Trigger: Always
- Question: "Which user scopes should be requested from the IDP?"
- Options:
	* `user_data + session_initial_prop` (DEFAULT) — `AddUserDataScope = True`, `AddSessionInitialPropertiesScope = True`
	* Custom — any subset of `user_data`, `user_additional_data`, `session_initial_prop`, `session_app_data`
- Phase: design

### DP-9: Impersonate
- Trigger: Always — ask before generating code
- Question: "Should users who log in via this auth type be linked to users of another auth type?"
- Options:
	* `none` (DEFAULT) — `!""` — this auth type manages its own independent user set; GAMRemote users are stored separately from any other type
	* `local` — on login GAM searches for a matching local (username/password) user by GUID → ExternalID → email; if found, the external login converges on that local user; external user data overwrites the stored values on each login
	* `<auth-type-name>` — links to any other existing auth type by its `Name`
- Impact: Non-empty value means logins via this auth type share user records with the target type. Use when one user must log in via multiple mechanisms (e.g., migrating from local to GAMRemote while keeping existing accounts). Leave empty when external users must be isolated from other auth types
- Phase: design

### DP-10: FunctionId
- Trigger: Always
- Question: "Should this auth type also synchronize roles from the external GAM IDP?"
- Options:
	* `AuthenticationAndRoles` (DEFAULT) — authenticates the user and retrieves roles from the external IDP; requires the IDP application to expose roles via the configured scope
	* `OnlyAuthentication` — authenticates only; roles are managed locally in this GAM repository and are not fetched from the IDP
- Impact: Use `OnlyAuthentication` when role management is purely local or when the IDP does not expose roles
- Phase: design

---

## Pattern 1 — Standard
Idempotent Load-or-New + Save/error shape — see [Idempotent Save Pattern](../../debugging/common-genexus-patterns.md#idempotent-save-pattern-load-or-new). Look up exact types/defaults on the `GAMAuthenticationTypeGAMRemote` EO (EO Verification Protocol)

Top-level: `Name` (DP-5), `IsEnable`, `FunctionId` (DP-10: `AuthenticationAndRoles` or `OnlyAuthentication`), `Description`, `Impersonate` (DP-9: empty = none/DEFAULT, or another auth type name to link users)

`GAMRemote` sub-SDT properties to set:
- `ClientId` / `ClientSecret` (DP-2: provided, or see Random credentials helper below)
- `RemoteServerURL` (DP-3, REQUIRED — external GAM IDP base URL)
- `SiteURL` (DP-4, REQUIRED — this SP's base URL, drives `redirect_uri`)
- `PKCEAuthentication.Enable` / `.Method` / `.LenghtChallenge` (DP-6)
- `SLOEnable` (DP-7)
- `AddUserDataScope` / `AddUserAdditionalDataScope` / `AddSessionInitialPropertiesScope` / `AddSessionApplicationDataScope` / `AdditionalScope` (DP-8)
- `AutovalidateExternalTokenAndRefresh`

---

## Pattern 2 — Encrypted variant (delta only)
Set `GAMRemote.RemoteServerKey` (32-char hex, DP-1 encrypted variant) in addition to Pattern 1. IDP application must have the matching key configured (Application → OAuth Authentication → RemoteServerKey). Name convention (DP-5): `gamremote-encrypted`

---

## Random credentials helper
`GAMHelper.GenerateSHA512(<name>)` (static method) derives deterministic values — call it once for `ClientId` (seed: name + `"_id"`) and once for `ClientSecret` (seed: name + `"_secret"`), idempotent across re-runs

---

## Validation rules on Save()
`Save()` sets `Success() = False` when:
- `Name` is empty
- `GAMRemote.ClientId` is empty
- `GAMRemote.ClientSecret` is empty
- `GAMRemote.RemoteServerURL` is empty
- `GAMRemote.SiteURL` is empty

Retrieve errors: `GetErrors()` → iterate `GAMErrorCollection` — see error block in Pattern 1

---

## GAMRemote as a chain hop
A GAMRemote auth type can be a hop in an IDP→IDP login chain: an upstream GAM IdP lists this type by name in a client application's `auth:` Local Login URL, so the upstream IdP redirects straight to the IdP this type targets (`RemoteServerURL`). That target IdP can repeat the pattern, extending the chain

- This type's `RemoteServerURL` defines the next IdP in the chain
- Login auto-redirect requires the matching OAuth 2.0/GAMRemote target to have `Redirect to Authenticate = True` at the upstream IdP
- For the chain concept, configuration, return path, and diagnostics see [IDP→IDP Login Chaining — `auth:` Local Login URL Proxy](../local-idp/idp-chaining.md)
