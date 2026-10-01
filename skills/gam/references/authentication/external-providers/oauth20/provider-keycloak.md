---
name: provider-keycloak
description: Keycloak OAuth 2.0 + OIDC — realm-based URL pattern, id_token_hint logout, and roles via token claim
---

# Provider — Keycloak (OAuth 2.0 + OIDC)
No built-in GAM template. Use `GAMInitAuthenticationTypeOAuth20.Default` as base and apply these overrides

Related files:
- [OAuth 2.0 Auth Type — Programmatic Initialization](provider-generic-oauth20.md) — initialization patterns and full property reference

---

## Endpoints
All Keycloak endpoints are realm-scoped. Replace `<host>` and `<realm>`

- Authorize — `https://<host>/realms/<realm>/protocol/openid-connect/auth`
- Token — `https://<host>/realms/<realm>/protocol/openid-connect/token`
- UserInfo — `https://<host>/realms/<realm>/protocol/openid-connect/userinfo`
- Discovery — `https://<host>/realms/<realm>/.well-known/openid-configuration`
- Signout — `https://<host>/realms/<realm>/protocol/openid-connect/logout`
- IssuerURL — `https://<host>/realms/<realm>`

## UserInfo field mapping
Standard OIDC claims:

```genexus
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserExternalId_Name = !"sub"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserEmail_Name = !"email"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserVerifiedEmail_Name = !"email_verified"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserName_Name = !"preferred_username"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserFirstName_Name = !"given_name"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserLastName_Name = !"family_name"
```

## Roles — via token claim, not a separate endpoint
Keycloak embeds roles in the ID/access token under the `realm_access.roles` nested claim. Configure the Roles sub-SDT to read from the UserInfo endpoint using the nested path:

```genexus
&GAMAuthenticationTypeOAuth20.OAuth20.Roles.URL = !"https://<host>/realms/<realm>/protocol/openid-connect/userinfo"
&GAMAuthenticationTypeOAuth20.OAuth20.Roles.ResponseRole_ExternalID_Name = !"realm_access.roles"
```

## Signout — id_token_hint required for session-scoped logout
Without `id_token_hint`, Keycloak terminates the entire realm SSO session (all apps). With it, only the current client session is terminated

```genexus
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.SLOEnable = True
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.URL = !"https://<host>/realms/<realm>/protocol/openid-connect/logout"
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.Method = GAMAccessMethod.GET
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.RedirectURL_Include = True
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.RedirectURL_Name = !"post_logout_redirect_uri"
```

To pass `id_token_hint`, include it via `AdditionalParameters` (value resolved at runtime by the application):

```genexus
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.AdditionalParameters = !"id_token_hint=<id-token>"
```
