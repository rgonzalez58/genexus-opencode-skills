---
name: domain-gamremoterest
description: GAM as SP — programmatic initialization of GAMAuthenticationTypeGAMRemoteRest (REST Password Grant against external GAM IDP) including Encrypted and 2FA variants
---

# GAM as SP — GAMRemoteRest (REST Password Grant) Initialization
Initializes `GAMAuthenticationTypeGAMRemoteRest` by code — configures a GAM SP to authenticate users via Password Grant (no browser) against an external GAM IDP, with optional 2FA

Related files:
- [GAM as REST IDP Server — Password Grant Flow](../local-idp/password-grant-flow.md) — runtime flow (IDP side)
- [GAM as SP — GAMRemote (Web Flow) Initialization](./domain-gamremote.md) — Web sibling (Authorization Code, PKCE, SLO)
- [GAM OTP & 2FA Encyclopedia](../local-idp/domain-otp-2fa.md) — OTP/TOTP auth type initialization (prerequisite for 2FA variants)
- [GAM Authentication Configuration Reference](../external-providers/common/domain-auth-config.md) — Backoffice auth-type configuration
- [GeneXus Patterns for GAM Integration](../../debugging/common-genexus-patterns.md) — error handling patterns
- [SSO REST — Single Sign-On for REST Services](../sso-rest/domain-sso-rest.md) — SSO REST token emission: when the IDP Application has `SSORESTEnable=True, SSORESTMode=Server`, this flow detects the SSO REST token in the IDP response and populates `&GAMSession.SSORestToken`

Variable types:
- `&GAMAuthenticationTypeGAMRemoteRest` — `GAMAuthenticationTypeGAMRemoteRest, GeneXusSecurity` (EO)
- `&GAMErrorCollection` — `GAMError, GeneXusSecurity` — Collection: True
- `&GAMError` — `GAMError, GeneXusSecurity`

---

## Decision Points
### DP-1: Variant
- Trigger: User requests GAMRemoteRest initialization
- Question: "Which variant?"
- Options:
	* `standard` (DEFAULT) — REST Password Grant, no extras
	* `encrypted` — adds `RemoteServerKey`; IDP Application must have the matching key configured
	* `2fa-otp` — adds OTP second factor (optional per user)
	* `2fa-otp-au` — adds OTP second factor forced for all users
	* `2fa-totp` — adds TOTP second factor
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

### DP-4: Auth type name
- Trigger: Always
- Question: "What name should this auth type have?"
- Options: free text
- DEFAULT: `gamremoterest` (+ suffix: `-encrypted`, `-2fa-otp`, `-2fa-otp-au`, `-2fa-totp`)
- Phase: design

### DP-5: 2FA auth type name
- Trigger: DP-1 = `2fa-otp`, `2fa-otp-au`, or `2fa-totp`
- Question: "What is the name of the existing OTP or TOTP auth type in this GAM repository?"
- Options: free text — must reference an existing auth type; see [GAM OTP & 2FA Encyclopedia](../local-idp/domain-otp-2fa.md)
- DEFAULT: `otp-2fa` (otp/otp-au variants) / `totp-2fa` (totp variant)
- Phase: design

### DP-6: Impersonate
- Trigger: Always — ask before generating code
- Question: "Should users who log in via this auth type be linked to users of another auth type?"
- Options:
	* `none` (DEFAULT) — `!""` — this auth type manages its own independent user set; GAMRemoteRest users are stored separately from any other type
	* `local` — on login GAM searches for a matching local (username/password) user by GUID → ExternalID → email; if found, the external login converges on that local user; external user data overwrites the stored values on each login
	* `<auth-type-name>` — links to any other existing auth type by its `Name`
- Impact: Non-empty value means logins via this auth type share user records with the target type. Use when one user must log in via multiple mechanisms (e.g., migrating from local to GAMRemoteRest while keeping existing accounts). Leave empty when external users must be isolated from other auth types
- Phase: design

### DP-7: FunctionId
- Trigger: Always
- Question: "Should this auth type also synchronize roles from the external GAM IDP?"
- Options:
	* `AuthenticationAndRoles` (DEFAULT) — authenticates the user and retrieves roles from the external IDP; requires the IDP application to expose roles via the configured scope
	* `OnlyAuthentication` — authenticates only; roles are managed locally in this GAM repository and are not fetched from the IDP
- Impact: Use `OnlyAuthentication` when role management is purely local or when the IDP does not expose roles
- Phase: design

---

## Pattern 1 — Standard
Idempotent Load-or-New + Save/error shape — see [Idempotent Save Pattern](../../debugging/common-genexus-patterns.md#idempotent-save-pattern-load-or-new). Look up exact types/defaults on the `GAMAuthenticationTypeGAMRemoteRest` EO (EO Verification Protocol)

Top-level: `Name` (DP-4), `IsEnable`, `FunctionId` (DP-7: `AuthenticationAndRoles` or `OnlyAuthentication`), `Description`, `Impersonate` (DP-6: empty = none/DEFAULT, or another auth type name to link users)

`GAMRemoteRest` sub-SDT properties to set:
- `ClientId` / `ClientSecret` (DP-2: provided, or see Random credentials helper below)
- `RemoteServerURL` (DP-3, REQUIRED — external GAM IDP base URL)
- `AddUserDataScope`, `AutovalidateExternalTokenAndRefresh`

---

## Pattern 2 — Encrypted variant (delta only)
Set `GAMRemoteRest.RemoteServerKey` (32-char hex, DP-1 encrypted variant) in addition to Pattern 1. IDP application must have the matching key configured (Application → OAuth Authentication → RemoteServerKey). Name convention (DP-4): `gamremoterest-encrypted`

---

## Pattern 3 — 2FA overlays (delta only)
Prerequisite: an OTP or TOTP auth type must already exist in this GAM — see [GAM OTP & 2FA Encyclopedia](../local-idp/domain-otp-2fa.md)

Set on `TwoFactorAuthentication` sub-SDT in addition to Pattern 1: `Enable = True`, `AuthenticationTypeName` (DP-5: name of the existing OTP/TOTP auth type), `FirstAuthenticationFactorExpiration` (seconds — `900` = 15 min default), `ForceForAllUsers`

- Pattern 3a — OTP optional per user (DP-1 `2fa-otp`): `AuthenticationTypeName = "otp-2fa"`, `ForceForAllUsers = False`. Name: `gamremoterest-2fa-otp`
- Pattern 3b — OTP forced for all (DP-1 `2fa-otp-au`): same as 3a with `ForceForAllUsers = True`. Name: `gamremoterest-2fa-otp-au`
- Pattern 3c — TOTP (DP-1 `2fa-totp`): `AuthenticationTypeName = "totp-2fa"`, `ForceForAllUsers = False`. Name: `gamremoterest-2fa-totp`

---

## Random credentials helper
`GAMHelper.GenerateSHA512(<name>)` (static method) derives deterministic values — call it once for `ClientId` (seed: name + `"_id"`) and once for `ClientSecret` (seed: name + `"_secret"`), idempotent across re-runs

---

## Validation rules on Save()
`Save()` sets `Success() = False` when:
- `Name` is empty
- `GAMRemoteRest.ClientId` is empty
- `GAMRemoteRest.ClientSecret` is empty
- `GAMRemoteRest.RemoteServerURL` is empty
- 2FA enabled and `TwoFactorAuthentication.AuthenticationTypeName` does not reference an existing auth type

Retrieve errors: `GetErrors()` → iterate `GAMErrorCollection` — see error block in Pattern 1
