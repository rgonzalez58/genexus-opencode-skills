---
name: domain-apikey
description: GAM APIkey — programmatic initialization of GAMAuthenticationTypeAPIkey (header/querystring auth, no browser, no redirect)
---

# GAM APIkey — Authentication Type Initialization
Initializes `GAMAuthenticationTypeAPIkey` by code — configures the API key authentication type for machine-to-machine access via key header or query string, with no browser redirect

Related files:
- [GAM as SP — GAMRemoteRest (REST Password Grant) Initialization](./domain-gamremoterest.md) — 2FA overlay shape (Pattern 3a/3b/3c) if wiring 2FA on APIkey
- [GAM OTP/TOTP — Authentication Type Initialization](./domain-otp.md) — prerequisite OTP/TOTP auth type if wiring 2FA
- [GeneXus Patterns for GAM Integration](../../debugging/common-genexus-patterns.md) — error handling patterns

Variable types:
- `&GAMAuthenticationTypeAPIkey` — `GAMAuthenticationTypeAPIkey, GeneXusSecurity` (EO)
- `&GAMErrorCollection` — `GAMError, GeneXusSecurity` — Collection: True
- `&GAMError` — `GAMError, GeneXusSecurity`
- `&Name` — `VarChar`

---

## Decision Points
### DP-1: Auth type name
- Trigger: Always
- Question: "What name should this auth type have?"
- Options: free text
- DEFAULT: `apikey-simple`
- Phase: design

### DP-2: Description
- Trigger: Always
- Question: "What description should this auth type have?"
- Options: free text
- DEFAULT: `APIkey Simple`
- Phase: design

---

## Pattern 1 — Standard
Idempotent Load-or-New + Save/error shape — see [Idempotent Save Pattern](../../debugging/common-genexus-patterns.md#idempotent-save-pattern-load-or-new). Look up exact types/defaults on the `GAMAuthenticationTypeAPIkey` EO (EO Verification Protocol)

Properties to set: `Name` (DP-1), `IsEnable`, `Description` (DP-2), `SmallImageName`, `BigImageName`, `TwoFactorAuthentication.Enable = False` (unless wiring 2FA — see below)

---

## 2FA support note
`GAMAuthenticationTypeAPIkey` exposes `TwoFactorAuthentication.*` (`Enable`, `AuthenticationTypeName`, `FirstAuthenticationFactorExpiration`, `ForceForAllUsers`). To wire 2FA on APIkey, follow the overlay shape in [GAM as SP — GAMRemoteRest (REST Password Grant) Initialization](./domain-gamremoterest.md) Pattern 3a/3b/3c — replace `&GAMAuthenticationTypeGAMRemoteRest` with `&GAMAuthenticationTypeAPIkey` and insert before `Save()` in Pattern 1 above. Dedicated 2FA patterns are out of scope for this file

---

## Validation rules on Save()
`Save()` sets `Success() = False` when:
- `Name` is empty

Retrieve errors: `GetErrors()` → iterate `GAMErrorCollection` — see error block in Pattern 1
