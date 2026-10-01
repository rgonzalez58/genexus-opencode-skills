---
name: password-grant-flow
description: GAM as REST IDP Server — OAuth 2.0 Password Grant flow on /oauth/gam/v2.0 (direct REST, no browser, mobile/IoT/server-to-server)
---

# GAM as REST IDP Server — Password Grant Flow
Pattern for any client (GeneXus or not) that authenticates against a GAM IDP using direct REST calls without browser or redirect. Uses `grant_type=password` (Resource Owner Password Credentials)

Related files:
- [GAM as Web IDP Server — Authorization Code Flow](authorization-code-flow.md) — Sibling pattern: GAM as Web IDP Server (Authorization Code, browser redirect)
- [GAM Authentication](../external-providers/common/domain-auth.md) — Internal GAMRemote flow (GAM client authenticating against external IDP)
- [OAuth User Scopes](scopes.md) — OAuth User Scopes returned in UserInfo
- [GAM Single Log Out (SLO)](../../logout/domain-slo.md) — Single Logout flows
- [GAM Authentication — Error Code Catalog](../external-providers/common/error-codes-auth.md) — Error codes for REST IDP responses

Official documentation: [HowTo: Use OAuth 2.0 Endpoints to authenticate with GAM as REST IDP Server](https://docs.genexus.com/en/wiki?55623)

---

## Architecture
```
┌──────────────────────┐         ┌───────────────────────┐
│  Any client          │         │  GAM IDP Server       │
│  (REST/Mobile/IoT/   │────────>│  /oauth/gam/v2.0/…    │
│   Script/Postman)    │<────────│  With GAM applied     │
│  No browser needed   │         │                       │
└──────────────────────┘         └───────────────────────┘
```

- The client can be any technology that makes HTTP requests — no GeneXus dependency
- No browser redirects — every call is a direct API call
- Uses `grant_type=password` (Resource Owner Password Credentials)
- Endpoints are `/oauth/gam/v2.0/…` (with the `/v2.0/` prefix)

CRITICAL: REST IDP endpoints are `/oauth/gam/v2.0/…`. Do NOT confuse with Web IDP endpoints on `/oauth/gam/…` — they are distinct flows

## Prerequisites on the GAM IDP (Backoffice)
Navigate to `Applications → Edit <app> → OAuth Authentication tab`

Under section header `REST services`:

- Check the master toggle for the target endpoint version:
	* `Allow OAuth 2.0 (password grant flow in /oauth/gam/v2.0):` — modern endpoint (recommended). API variable `ClientAllowRemoteRESTAuthentication`
	* `Allow OAuth 2.0 (password grant flow in /oauth):` — legacy REST v1.0 endpoint (backward compatibility). API variable `ClientAllowRESTv10Authentication`
- In the `Allowed user scopes (SSO REST, Mini App, API key, password grant flow)` grid, check the scopes the client is allowed to request:
	* `User data` → scope `gam_user_data`
	* `User additional data` → scope `gam_user_additional_data`
	* `User roles` → scope `gam_user_roles`
	* `Session initial properties` → scope `gam_session_initial_prop`
	* `Session application data` → scope `gam_session_app_data`
- Fill `Additional REST scopes` with custom scope names (optional)
- Check `Single-user access using OAuth 2.0` to enforce one access token per user (optional)
- Check `Authentication request must include user scopes?` to force the client to send `scope=…` explicitly (optional)
- Check `Do not share user IDs` to omit internal user identifiers from tokens (optional)
- Fill `Repository GUID` when the IDP serves multiple tenants (multi-tenant only)
- Use `GENERATE KEY` under `Private encryption key` to rotate the token encryption key (optional)
- Configure `Maximum OAuth token renewals` in Security Policies to enable refresh_token (default disables refresh)
- Confirm the application — GAM generates `Client Id` and `Client Secret` on the General tab

Field-to-API mapping:
- `Allow OAuth 2.0 (password grant flow in /oauth/gam/v2.0):` ↔ `ClientAllowRemoteRESTAuthentication`
- `Allow OAuth 2.0 (password grant flow in /oauth):` ↔ `ClientAllowRESTv10Authentication`
- `User data` / `User roles` / … (scope grid) ↔ `ClientAllowGet*REST` (for example `ClientAllowGetUserDataREST`, `ClientAllowGetUserRolesREST`)
- `Authentication request must include user scopes?` ↔ `ClientAuthenticationRequestMustIncludeUserScopes`

## Complete Flow (3 steps + refresh)
```
Step 1: POST /oauth/gam/v2.0/access_token (username + password → access_token)
	↓
Step 2: GET /oauth/gam/v2.0/userinfo (Bearer token → user data)
	↓ (when the token expires → HTTP 401 code 103)
Step 3: GET /oauth/gam/v2.0/access_token (refresh_token → new access_token)
```

Key difference from Web IDP — there is no signin/redirect step. The client sends username/password directly to the access_token endpoint

## Step 1 — Access Token (Password Grant)
Endpoint: `POST https://<idp_domain>/<virtual_dir>/oauth/gam/v2.0/access_token`

Headers:
- `Content-Type: application/x-www-form-urlencoded`

Required parameters:
- `client_id` = `<Client_ID>` — ID of the application registered on the IDP
- `client_secret` = `<Client_Secret>` — application secret
- `grant_type` = `password` — Resource Owner Password Credentials
- `scope` = `gam_user_data+gam_user_roles` — recommended default. See `../scopes.md`
- `username` = `<user>` — user credentials
- `password` = `<password>` — user credentials

Optional parameters:
- `authentication_type_name` — name of the auth type (for example `local`); defaults to Repository default
- `initial_properties` — custom user properties as JSON array; default empty
- `repository` — required only for multi-tenant IDPs; default `default`
- `request_token_type` — `OAuth` or `Web`; selects which Security Policy applies. Default `OAuth`
- `additional_parameters` — additional user information; default empty

Format for `initial_properties`:

```json
[{"Id":"Company","Value":"GeneXus"},{"Id":"Branch","Value":"Uruguay"}]
```

Successful response (HTTP 200):

```json
{
	"access_token": "<access_token>",
	"token_type": "Bearer",
	"expires_in": 0,
	"refresh_token": "",
	"scope": "gam_user_data+gam_user_roles+session_initial_prop",
	"user_guid": "736c85fa-5123-437d-a528-93471d3bae42"
}
```

Notes:
- `refresh_token` is empty unless `Maximum OAuth token renewals` is configured in Security Policies
- `expires_in: 0` indicates the token does not expire automatically (subject to Security Policy)

Example with curl:

```bash
curl -X POST "https://my-idp.com/MyApp/oauth/gam/v2.0/access_token" \
	-H "Content-Type: application/x-www-form-urlencoded" \
	-d "client_id=<CLIENT_ID>" \
	-d "client_secret=<CLIENT_SECRET>" \
	-d "grant_type=password" \
	-d "scope=gam_user_data+gam_user_roles" \
	-d "username=admin" \
	-d "password=<admin-password>"
```

GeneXus example (HttpClient):

```genexus
&HttpClient = new()
&HttpClient.Host = &IDPHost
&HttpClient.BaseUrl = !"oauth/gam/v2.0"
&HttpClient.Secure = 1
&HttpClient.AddVariable(!"client_id", &ClientId)
&HttpClient.AddVariable(!"client_secret", &ClientSecret)
&HttpClient.AddVariable(!"grant_type", !"password")
&HttpClient.AddVariable(!"scope", !"gam_user_data+gam_user_roles")
&HttpClient.AddVariable(!"username", &Username)
&HttpClient.AddVariable(!"password", &Password)
&HttpClient.Execute(!"POST", !"access_token")

If &HttpClient.StatusCode = 200
	&Oauth20AccessTokenSDT.FromJson(&HttpClient.ToString())
EndIf
```

JavaScript example (fetch):

```javascript
const response = await fetch(
	`https://my-idp.com/MyApp/oauth/gam/v2.0/access_token`,
	{
		method: 'POST',
		headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
		body: new URLSearchParams({
			client_id: CLIENT_ID,
			client_secret: CLIENT_SECRET,
			grant_type: 'password',
			scope: 'gam_user_data+gam_user_roles',
			username: user,
			password: pass
		})
	}
);
const tokenData = await response.json();
```

Python example (requests):

```python
import requests

response = requests.post(
	f"https://my-idp.com/MyApp/oauth/gam/v2.0/access_token",
	data={
		"client_id": CLIENT_ID,
		"client_secret": CLIENT_SECRET,
		"grant_type": "password",
		"scope": "gam_user_data+gam_user_roles",
		"username": user,
		"password": password
	}
)
token_data = response.json()
```

## Step 2 — UserInfo
Endpoint: `GET https://<idp_domain>/<virtual_dir>/oauth/gam/v2.0/userinfo`

Headers:
- `Content-Type: application/x-www-form-urlencoded`
- `Authorization: Bearer <access_token>`

NOTE: Unlike Web IDP which uses `Authorization: <token>`, REST IDP uses `Authorization: Bearer <token>` with the explicit Bearer prefix

Successful response (HTTP 200):

```json
{
	"guid": "492c664b-8831-4efb-8618-0c8e86e75446",
	"username": "admin",
	"email": "admin@example.com",
	"verified_email": true,
	"first_name": "Administrator",
	"last_name": "User",
	"external_id": "",
	"birthday": "2000-01-01",
	"gender": "N",
	"url_image": "https://",
	"url_profile": "",
	"phone": "+598",
	"address": ".",
	"city": ".",
	"state": ".",
	"post_code": ".",
	"language": "Eng",
	"timezone": ".",
	"application_data": "",
	"CustomInfo": "",
	"roles": ["is_gam_administrator"]
}
```

NOTE: The REST IDP response includes `roles` and `application_data` — absent in Web IDP responses — because the default scope includes `gam_user_roles`

Example with curl:

```bash
curl -X GET "https://my-idp.com/MyApp/oauth/gam/v2.0/userinfo" \
	-H "Content-Type: application/x-www-form-urlencoded" \
	-H "Authorization: Bearer <access_token>"
```

GeneXus example:

```genexus
&HttpClient = new()
&HttpClient.Host = &IDPHost
&HttpClient.BaseUrl = !"oauth/gam/v2.0"
&HttpClient.Secure = 1
&HttpClient.AddHeader(!"Authorization", !"Bearer " + &Oauth20AccessTokenSDT.access_token)
&HttpClient.Execute(!"GET", !"userinfo")

If &HttpClient.StatusCode = 200
	&UserInfoSDT.FromJson(&HttpClient.ToString())
EndIf
```

## Step 3 — Refresh Token
Endpoint: `GET https://<idp_domain>/<virtual_dir>/oauth/gam/v2.0/access_token`

NOTE: Refresh in REST IDP uses GET (not POST like Web IDP)

Query string parameters:
- `client_id` = `<Client_ID>`
- `client_secret` = `<Client_Secret>`
- `grant_type` = `refresh_token`
- `refresh_token` = `<refresh_token>` (from Step 1)

Successful response (HTTP 200):

```json
{
	"access_token": "<access_token>",
	"token_type": "Bearer",
	"expires_in": 6000,
	"refresh_token": "<refresh_token>",
	"scope": "gam_user_data+gam_user_additional_data+gam_session_initial_prop+gam_user_roles",
	"user_guid": "492c664b-8831-4efb-8618-0c8e86e75446"
}
```

When to refresh:
- API call returns HTTP 401 with `{"error": {"code": "103", "message": "Token expired, log in again."}}`
- If no refresh_token, re-authenticate with username/password (Step 1)

Example with curl:

```bash
curl -X GET "https://my-idp.com/MyApp/oauth/gam/v2.0/access_token?\
client_id=<CLIENT_ID>&\
client_secret=<CLIENT_SECRET>&\
grant_type=refresh_token&\
refresh_token=<REFRESH_TOKEN>" \
	-H "Content-Type: application/x-www-form-urlencoded"
```

## Advanced Parameters
`request_token_type` — OAuth vs Web:
- `OAuth` (default) — applies OAuth Security Policy from IDP; use for REST clients, mobile, scripts
- `Web` — applies Web Session Security Policy from IDP; use when the client wants to behave like a web session

`authentication_type_name`:
- Allows specifying which authentication type to use when the IDP Repository has multiple auth types configured. Defaults to the Repository default

`repository` — Multi-tenant:
- Required only when the GAM IDP handles multiple repositories. Identifies which tenant to authenticate against

## Endpoints Summary
- Step 1 — `/oauth/gam/v2.0/access_token` (POST) — obtain token with username/password
- Step 2 — `/oauth/gam/v2.0/userinfo` (GET) — get authenticated user data
- Step 3 — `/oauth/gam/v2.0/access_token` (GET) — refresh token when access_token expires

## GAM External Objects (for GeneXus clients)
- `GAMOAuth20AccessToken` — SDT for the access_token endpoint response
- `GAMOAuth20UserInfo` — SDT for the userinfo endpoint response

---

## Web IDP vs REST IDP — When to use each
- Official documentation — Web IDP = [wiki 49817](https://docs.genexus.com/en/wiki?49817); REST IDP = [wiki 55623](https://docs.genexus.com/en/wiki?55623)
- Endpoints — Web IDP = `/oauth/gam/…`; REST IDP = `/oauth/gam/v2.0/…`
- OAuth flow — Web IDP = Authorization Code (redirect); REST IDP = Password Grant (direct)
- Requires browser — Web IDP = yes (redirect to IDP login); REST IDP = no (pure API)
- `grant_type` — Web IDP = `authorization_code`; REST IDP = `password`
- Signin step — Web IDP = `/oauth/gam/signin` redirect; REST IDP = none, direct POST
- Auth header (userinfo) — Web IDP = `Authorization: <token>`; REST IDP = `Authorization: Bearer <token>`
- Default scope — Web IDP = `gam_user_data`; REST IDP = `gam_user_data+gam_user_roles`
- UserInfo response — Web IDP = without roles; REST IDP = includes `roles` and `application_data`
- PKCE — Web IDP = supported; REST IDP = not applicable (no code exchange)
- Refresh method — Web IDP = POST; REST IDP = GET
- Typical use case — Web IDP = web apps with login form on the IDP; REST IDP = mobile, scripts, IoT, server-to-server integrations
- Security — Web IDP = more secure (credentials never pass through the client); REST IDP = less secure (client handles user/pass directly)
- Backoffice config — Web IDP = `Allow OAuth 2.0 (authorization code flow in /oauth/gam)`; REST IDP = `Allow OAuth 2.0 (password grant flow in /oauth/gam/v2.0)`
