---
name: domain-local
description: GAM Local — programmatic 2FA wiring on GAMAuthenticationTypeLocal (the auto-provisioned 'local' row): OTP optional, OTP forced for all, TOTP
---

# GAM Local — 2FA Wiring on Local Authentication Type
Wires 2FA on `GAMAuthenticationTypeLocal` by code — the `local` row is auto-provisioned by GAM at repository creation; this file only configures OTP or TOTP as the second factor on that existing row

Related files:
- [GAM OTP/TOTP — Authentication Type Initialization](./domain-otp.md) — create the OTP/TOTP auth type referenced in DP-2
- [GAM OTP & 2FA Encyclopedia](../local-idp/domain-otp-2fa.md) — runtime 2FA flow, enrollment, Backoffice configuration, error codes
- [GeneXus Patterns for GAM Integration](../../debugging/common-genexus-patterns.md) — error handling patterns

Variable types:
- `&GAMAuthenticationTypeLocal` — `GAMAuthenticationTypeLocal, GeneXusSecurity` (EO)
- `&GAMErrorCollection` — `GAMError, GeneXusSecurity` — Collection: True
- `&GAMError` — `GAMError, GeneXusSecurity`

---

## Decision Points
### DP-1: 2FA variant
- Trigger: User requests Local 2FA initialization
- Question: "Which 2FA variant for Local?"
- Options:
	* `2fa-otp` (DEFAULT) — OTP second factor, optional per user
	* `2fa-otp-au` — OTP second factor, forced for all users
	* `2fa-totp` — TOTP second factor, optional per user
- Phase: design

### DP-2: 2FA auth type name
- Trigger: Always
- Question: "What is the name of the existing OTP or TOTP auth type in this GAM repository?"
- Options: free text — must reference an existing auth type; see [GAM OTP/TOTP — Authentication Type Initialization](./domain-otp.md)
- DEFAULT: `otp-2fa` (DP-1 `2fa-otp` / `2fa-otp-au`) / `totp-2fa` (DP-1 `2fa-totp`)
- Phase: design

### DP-3: First-factor expiration
- Trigger: Always
- Question: "How long (in seconds) before the first-factor credential expires while awaiting the OTP step?"
- Options: numeric value in seconds
- DEFAULT: `900` (15 minutes)
- Phase: design

---

## Pattern 1 — 2FA OTP optional (DP-1: `2fa-otp`)
Loads the auto-provisioned `local` row (`Load("local")`, no `= new()` — the row always exists) and sets `TwoFactorAuthentication` sub-SDT properties: `Enable = True`, `AuthenticationTypeName` (DP-2, prerequisite: an OTP auth type must already exist — see [GAM OTP/TOTP — Authentication Type Initialization](./domain-otp.md)), `FirstAuthenticationFactorExpiration` (DP-3, seconds — `900` = 15 min default), `ForceForAllUsers = False`. Save/error — see [Idempotent Save Pattern](../../debugging/common-genexus-patterns.md#idempotent-save-pattern-load-or-new). Look up exact types on the `GAMAuthenticationTypeLocal` EO (EO Verification Protocol)

---

## Pattern 2 — 2FA OTP forced for all users (delta only)
Same as Pattern 1 with `ForceForAllUsers = True` (DP-1: `2fa-otp-au`)

---

## Pattern 3 — 2FA TOTP (delta only)
Same as Pattern 1 with `AuthenticationTypeName` pointing to an existing TOTP auth type (DP-1: `2fa-totp`; prerequisite — see [GAM OTP/TOTP — Authentication Type Initialization](./domain-otp.md) Pattern 2)

---

## Validation rules on Save()
`Save()` sets `Success() = False` when:
- `Load("local")` does not find the row — the `local` auth type does not exist; run repository initialization first
- `TwoFactorAuthentication.AuthenticationTypeName` does not reference an existing auth type in this GAM repository

Retrieve errors: `GetErrors()` → iterate `GAMErrorCollection` — see error block in Pattern 1
