---
name: provider-auth0
description: Auth0 OAuth 2.0 + OIDC — tenant-based URLs, audience parameter for JWT, returnTo logout
---

# Provider — Auth0 (OAuth 2.0 + OIDC)
No built-in GAM template. Use `GAMInitAuthenticationTypeOAuth20.Default` as base and apply these overrides

Related files:
- [OAuth 2.0 Auth Type — Programmatic Initialization](provider-generic-oauth20.md) — initialization patterns and full property reference

---

## Endpoints
Replace `<tenant>` with your Auth0 tenant domain (e.g. `myapp.us.auth0.com`)

- Authorize — `https://<tenant>/authorize`
- Token — `https://<tenant>/oauth/token`
- UserInfo — `https://<tenant>/userinfo`
- Discovery — `https://<tenant>/.well-known/openid-configuration`
- Signout — `https://<tenant>/v2/logout`
- IssuerURL — `https://<tenant>/` (trailing slash required)

## UserInfo field mapping
```genexus
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserExternalId_Name = !"sub"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserEmail_Name = !"email"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserVerifiedEmail_Name = !"email_verified"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserName_Name = !"nickname"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserFirstName_Name = !"given_name"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserLastName_Name = !"family_name"
&GAMAuthenticationTypeOAuth20.OAuth20.UserInfo.ResponseUserURLImage_Name = !"picture"
```

## audience parameter — required when issuing API JWTs
Auth0 issues opaque access tokens by default. To get a signed JWT bound to an API, add `audience` to the Authorize request:

```genexus
&GAMAuthenticationTypeOAuth20.OAuth20.Authorize.AdditionalParameters = !"audience=https://myapi.example.com"
```

Register the audience value as an API identifier in the Auth0 Dashboard → APIs

## Signout — returnTo instead of post_logout_redirect_uri
Auth0 uses `returnTo` (not the standard `post_logout_redirect_uri`) and requires `client_id`:

```genexus
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.SLOEnable = True
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.URL = !"https://<tenant>/v2/logout"
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.Method = GAMAccessMethod.GET
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.ClientId_Include = True
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.ClientId_Name = !"client_id"
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.RedirectURL_Include = True
&GAMAuthenticationTypeOAuth20.OAuth20.Signout.RedirectURL_Name = !"returnTo"
```

Register the logout return URL under Allowed Logout URLs in Auth0 Dashboard → Applications → Settings
