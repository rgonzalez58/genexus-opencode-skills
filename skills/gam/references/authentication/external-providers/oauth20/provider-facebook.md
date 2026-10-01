---
name: provider-facebook
description: Facebook Login — dedicated GAMAuthenticationTypeFacebook EO (preferred), and Graph API fields param when using full OAuth 2.0
---

# Provider — Facebook
Related files:
- [OAuth 2.0 Auth Type — Programmatic Initialization](provider-generic-oauth20.md) — full OAuth 2.0 reference (Option B)

**Option A and Option B use different GAM entity types and cannot be combined.** Choose one per auth type configuration: either `GAMAuthenticationTypeFacebook` (built-in) or `GAMAuthenticationTypeOAuth20` (generic)

---

## Option A — Dedicated EO (preferred)
GAM ships a dedicated External Object for Facebook: `GAMAuthenticationTypeFacebook, GeneXusSecurity`. It is **not** `GAMAuthenticationTypeOAuth20` — it has its own simplified property tree. Variable type: `exo:GAMAuthenticationTypeFacebook, GeneXusSecurity`

Idempotent Load-or-New + Save/error shape — see [Idempotent Save Pattern](../../../debugging/common-genexus-patterns.md#idempotent-save-pattern-load-or-new). Look up exact types/defaults on the EO (EO Verification Protocol)

Properties to set: `Name`, `IsEnable`, `Description`, `Impersonate`, and on the `Facebook` sub-SDT: `ClientId`, `ClientSecret`, `SiteURL` (**must be HTTPS** — Facebook rejects `http://` redirect URIs except `localhost`)

## Option B — Full OAuth 2.0 (when you need scope/field control)
Use `GAMAuthenticationTypeOAuth20` (Pattern 2 from `provider-generic-oauth20.md`) with these Facebook-specific overrides:

- Endpoints (bump `v18.0` to the current Graph API version): Authorize `https://www.facebook.com/v18.0/dialog/oauth`, Token `https://graph.facebook.com/v18.0/oauth/access_token`, UserInfo `https://graph.facebook.com/v18.0/me`
- `UserInfo.AdditionalParameters` MUST list every field explicitly (`fields=id,email,name,first_name,last_name,picture`) — Facebook's `/me` endpoint returns only `id` and `name` by default
- `UserInfo` response-mapping properties map to Facebook's field names: `ResponseUserExternalId_Name`→`id`, `ResponseUserEmail_Name`→`email`, `ResponseUserName_Name`→`name`, `ResponseUserFirstName_Name`→`first_name`, `ResponseUserLastName_Name`→`last_name`, `ResponseUserURLImage_Name`→`picture.data.url`
- `Authorize.Scope_Value` = `email public_profile`
- Facebook does not support OIDC — leave `OpenIDConnectAuthentication.Enable = False`
