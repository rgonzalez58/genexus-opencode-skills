---
name: gamsecuritypolicy-code-provider
description: Capabilities and quirks for idempotent initialization of GAMSecurityPolicy. Idempotency strategy B — Id lookup. EO is source of truth for properties and methods
---

# GAMSecurityPolicy Code Provider
**EO**: `GAMSecurityPolicy` in `ref/GeneXusSecurity/` — locate by content header, not by an assumed file-name suffix (see [global-constraints.md § Where GAM's EOs live](../../../global-constraints.md#where-gams-eos-live))
Derive the SDT using the [SDT Generation Workflow](../../../global-constraints.md#sdt-generation-workflow--derive-the-sdt-11-from-the-eo)

---

## What you can use
- **Idempotency strategy**: B — Id lookup. `&GAMSecurityPolicy.Load(Id)` → check `Success()`. `Id` is read-only and assigned by GAM — never set or hardcode it on a new policy; `new()` (when reached) relies on GAM's own assignment, not a developer-supplied value
- **Capabilities**: load/save by Id — read the EO for exact method names and parameters
- **Read/Write properties**: derived from EO — covers name, web session timeout, OAuth token expiry and max renovations, refresh token expiry, access code expiry, password period, password history size, minimum password length, and more. Open the EO file for the complete list
- **Time units**: all timeout/expiry properties are in **minutes**. Example: `OAuthRefreshTokenExpire = 525600` = 1 year
- **Password properties**: `PeriodChangePassword = 0` disables mandatory rotation; `MaximumPasswordHistoryEntries` prevents reuse

## Known facts (not derivable from the EO)
- Policy `Id = 1` is the default policy pre-created by GAM on first build — this is a read (a known constant passed to `Load`), never a value assigned by the developer. Strategy B on `Id = 1` always finds it — the `new()` fallback is rarely reached in practice
- **Timeout Conflict rule**: `OAuthTokenExpire` MUST be ≥ Web Server Session Timeout (IIS/Tomcat). Violation causes Code 17/114 intermittently after login

## Cross-references
- EO: `GAMSecurityPolicy` in `ref/GeneXusSecurity/`
- [SDT Generation Workflow](../../../global-constraints.md#sdt-generation-workflow--derive-the-sdt-11-from-the-eo)
- [GAM Entity Initialization — Declarative Pattern](../entity-initialization.md)
- [GAM Entity Initialization — Consolidated Pattern](../entity-initialization-consolidated.md) (default pattern)
- [Security policies domain](../../../authorization/domain-security-policies.md)
- [Backoffice: security policies](../../../backoffice/security-policies.md)
