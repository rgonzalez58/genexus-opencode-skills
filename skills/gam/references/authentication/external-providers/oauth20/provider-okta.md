---
name: provider-okta
description: Okta OAuth 2.0 + OIDC — specific endpoints, OIDC discovery, revoke-based signout, and custom authorization server pattern
---

# Provider — Okta (OAuth 2.0 + OIDC)
No built-in GAM template. Use `GAMInitAuthenticationTypeOAuth20.Default` as base and apply these overrides

Related files:
- [OAuth 2.0 Auth Type — Programmatic Initialization](provider-generic-oauth20.md) — initialization patterns and full property reference

---

## Endpoints
Replace `<domain>` with your Okta tenant domain (e.g. `acme.okta.com`)

- Authorize — `https://<domain>/oauth2/v1/authorize`
- Token — `https://<domain>/oauth2/v1/token`
- UserInfo — `https://<domain>/oauth2/v1/userinfo`
- Discovery — `https://<domain>/.well-known/openid-configuration`
- Signout — `https://<domain>/oauth2/v1/revoke`

**Custom authorization server:** replace `/oauth2/v1/` with `/oauth2/<auth-server-id>/v1/`

## UserInfo field mapping
```genexus
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserExternalId_Name = !"sub"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserEmail_Name = !"email"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserVerifiedEmail_Name = !"email_verified"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserName_Name = !"preferred_username"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserFirstName_Name = !"given_name"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserLastName_Name = !"family_name"
```

## OIDC
```genexus
&GAMAuthenticationTypeOAuth20.OAuth20.Authorize.OpenIDConnectAuthentication.Enable = True
&GAMAuthenticationTypeOAuth20.OAuth20.Authorize.OpenIDConnectAuthentication.ValidIDToken = True
&GAMAuthenticationTypeOAuth20.OAuth20.Authorize.OpenIDConnectAuthentication.UseDiscoveryURL = True
&GAMAuthenticationTypeOAuth20.OAuth20.Authorize.OpenIDConnectAuthentication.DiscoveryURL = !"https://<domain>/.well-known/openid-configuration"
&GAMAuthenticationTypeOAuth20.OAuth20.Authorize.OpenIDConnectAuthentication.IssuerURL = !"https://<domain>"
```

Scope: `openid email profile`. Add `offline_access` for refresh tokens

## Signout — Okta uses a revoke endpoint (POST), not a redirect
```genexus
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.SLOEnable = True
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.URL = !"https://<domain>/oauth2/v1/revoke"
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.Method = GAMAccessMethod.POST
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.Header_AuthorizationBasic_Include = True
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.AccessToken_Include = True
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.AccessToken_Name = !"token"
```
