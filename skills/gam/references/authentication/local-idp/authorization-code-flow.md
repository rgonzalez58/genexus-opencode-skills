---
name: authorization-code-flow
description: GAM as Web IDP Server — OAuth 2.0 Authorization Code flow on /oauth/gam (browser redirect, PKCE, SLO handler for non-GAM clients)
---

# GAM as Web IDP Server — Authorization Code Flow
Pattern for applications (GeneXus or not) that authenticate against a GAM IDP via Web using the OAuth 2.0 Authorization Code flow with browser redirect

Related files:
- [GAM as REST IDP Server — Password Grant Flow](password-grant-flow.md) — Sibling pattern: GAM as REST IDP Server (Password Grant, no browser)
- [GAM Authentication](../external-providers/common/domain-auth.md) — Internal GAMRemote flow (GAM client authenticating against external IDP)
- [OAuth User Scopes](scopes.md) — OAuth User Scopes returned in UserInfo
- [GAM Single Log Out (SLO)](../../logout/domain-slo.md) — Single Logout flows
- [GAM Authentication — Error Code Catalog](../external-providers/common/error-codes-auth.md) — Error codes for Web IDP responses

Official documentation: [GAM - OAuth 2.0 Endpoints to use GAM as Web IDP Server](https://docs.genexus.com/en/wiki?49817)

When GAM is the client authenticating against an external IDP (instead of serving as IDP), see "OAuth 2.0 (External IDP Authentication)" in `../domain-auth.md`

---

## Architecture
```
┌─────────────────────┐        ┌──────────────────────┐
│  Client Application │        │  GAM IDP Server      │
│  (with or w/o GAM)  │───────>│  /oauth/gam/…        │
│  Browser redirect   │<───────│  With GAM applied    │
└─────────────────────┘        └──────────────────────┘
```

CRITICAL: Web IDP endpoints are `/oauth/gam/…` (without `/v2.0/`). Do NOT confuse with REST IDP endpoints on `/oauth/gam/v2.0/…` — they are distinct flows

## Prerequisites on the GAM IDP (Backoffice)
Navigate to `Applications → Edit <app> → OAuth Authentication tab`

Under section header `WEB (IDP using SSO, authorization code flow)`:

- Check `Allow OAuth 2.0 (authorization code flow in /oauth/gam):` — master toggle that enables the Web IDP endpoints (API variable `ClientAllowRemoteAuthentication`)
- Set `OAuth 2.0 Option` dropdown:
	* `Basic` — OAuth without PKCE
	* `Basic and PKCE` — OAuth with optional PKCE (backward compatible)
	* `PKCE` — PKCE required
- Set `Allowed PKCE Method` dropdown (only if PKCE enabled): `S256` or `plain`
- In the `Allowed user scopes (SSO REST, Mini App, API key, password grant flow)` grid, check the scopes the client is allowed to request:
	* `User data`
	* `User additional data`
	* `User roles`
	* `Session initial properties`
	* `Session application data`
- Fill `Additional user scopes` with custom scope names (optional)
- Fill `Local Login URL` — MANDATORY, the Backoffice refuses to save the application without it. See "Local Login URL — GAMExampleIDPLogin" below
- Fill `Callback URLs` with the client's callback endpoint(s)
- Check `Custom callback URL?` only for non-GAM clients (GAM clients auto-build `<BaseURL>/oauth/gam/callback`)
- Set `State parameter name` if the default `state` must be overridden
- For SLO, fill `Custom Single Logout URLs` and `Valid URLs after Single Logout`
- Confirm the application — GAM generates `Client Id` and `Client Secret` on the General tab

Field-to-API mapping:
- `Allow OAuth 2.0 (authorization code flow in /oauth/gam):` ↔ `ClientAllowRemoteAuthentication`
- `OAuth 2.0 Option` ↔ `ClientAllowRemoteAuthenticationOAuth20Option`
- `Local Login URL` ↔ `ClientLocalLoginURL`
- `Custom callback URL?` ↔ `ClientCallbackURLisCustom`

## Local Login URL — `GAMExampleIDPLogin`
`Local Login URL` (`ClientLocalLoginURL`) is the login page the IDP renders when a client hits `/oauth/gam/signin`. It is MANDATORY — the Backoffice refuses to save the application while it is empty

GAM ships a dedicated login object for the IDP role: `GAMExampleIDPLogin`. This is the object to use whenever GAM acts as Web IDP Server. Do NOT point this field at `GAMExampleLogin` — that object is the login for the application's own web sessions, not for the IDP signin flow

Value format — `<base_url>` + the deployed object name, generator-dependent:

- .NET / .NET Core — `<base_url>/gamexampleidplogin.aspx`
- Java — `<base_url>/<package>.gamexampleidplogin` (the package is the one configured in `Applications → Configuration > Environment → Package`)

Examples:

```
http://localhost/KB_IDP_DEMO/gamexampleidplogin.aspx
http://my-idp.com:8080/MyApp/servlet/com.mycompany.myapp.gamexampleidplogin
```

Alternatives for this same field:
- A custom login WebPanel URL, when the IDP login page must be branded or extended
- `auth:<authtype>[;…]`, to skip the login page entirely and proxy to another IdP — see [IDP→IDP Login Chaining — `auth:` Local Login URL Proxy](idp-chaining.md)

Common errors:
- Field left empty — the application cannot be saved
- Pointing at `GAMExampleLogin` instead of `GAMExampleIDPLogin` — the signin flow lands on the wrong login object and the OAuth context is lost
- Wrong generator format (`.aspx` on a Java deployment, or a missing package prefix) — the redirect resolves to a 404
- Host/virtual directory that does not match the actual deployment — the browser redirect fails before the user can authenticate

## Complete Flow (4 steps)
```
Step 1: Browser → /oauth/gam/signin (redirect with client_id, state, redirect_uri)
	↓ User authenticates at the IDP
Step 2: IDP redirects to callback with ?code=…&state=…
	↓ Server-to-server
Step 3: POST /oauth/gam/access_token (exchange code for access_token)
	↓
Step 4: GET /oauth/gam/userinfo (get authenticated user data)
	↓ (optional, when the token expires)
Step 5: POST /oauth/gam/access_token with grant_type=refresh_token
```

## Step 1 — Signin (Redirect to IDP)
Endpoint: `GET https://<idp_domain>/<virtual_dir>/oauth/gam/signin`

The client redirects the user's browser to this endpoint to start authentication

Required parameters:
- `response_type` = `code` — if absent, GAM treats the request as `oauth=auth`
- `client_id` = `<Client_ID>` — ID of the application registered on the IDP
- `redirect_uri` = `<callback_url>` — must match the Callback URL configured in Backoffice
- `state` = `<random_string>` — random string to validate the callback (anti-CSRF)

Optional parameters:
- `scope` — required when `Authentication request must include user scopes?` is enabled. Example: `gam_user_data`
- `authentication_type_name` — required when multiple auth types are configured
- `code_challenge_method` — required when using PKCE. `S256` or `PLAIN`
- `code_challenge` — required when using PKCE. Randomly generated value

URL example:

```
https://<idp_domain>/<virtual_dir>/oauth/gam/signin?
	response_type=code&
	scope=gam_user_data&
	client_id=<Client_ID>&
	redirect_uri=https://<your_server>/<virtual_directory>/oauth/callback&
	state=<random_alphanumeric>
```

Successful response — the IDP redirects to `redirect_uri`:

```
https://<your_server>/<virtual_directory>/oauth/callback?
	code=<authorization_code>&
	state=<random_alphanumeric>
```

Error response — the IDP redirects to `redirect_uri` with error parameters:

```
https://<your_server>/<virtual_directory>/oauth/callback?
	state=<random_alphanumeric>&
	error_code=<GAM_Error_Code>&
	error_message=<GAM_Error_Message>
```

CRITICAL: The client MUST validate that the received `state` matches the one sent. If it does not match, reject the callback as possible CSRF

GeneXus — build the redirect:

```genexus
// Web Panel Event Start or "Login with IDP" button
&State = GAMGenerateRandomString(32)
&WebSession.Set(!"OAuthState", &State)

&SigninURL = &IDPURL.Trim()
&SigninURL += !"/oauth/gam/signin"
&SigninURL += !"?response_type=code"
&SigninURL += !"&client_id=" + &ClientId.Trim()
&SigninURL += !"&redirect_uri=" + UrlEncode(&CallbackURL)
&SigninURL += !"&state=" + &State
&SigninURL += !"&scope=gam_user_data"

Link(Format(!"%1", &SigninURL))
```

## Step 2 — Access Token (Server-to-server)
Endpoint: `POST https://<idp_domain>/<virtual_dir>/oauth/gam/access_token`

After receiving the `code` in the callback, the server exchanges the code for an access_token. This request is server-to-server, not from the browser

Headers:
- `Content-Type: application/x-www-form-urlencoded`

Body parameters:
- `grant_type` = `authorization_code`
- `code` = `<code_from_callback>` — received in Step 1
- `client_id` = `<Client_ID>`
- `client_secret` = `<Client_Secret>` — NOT required when PKCE is used; see [PKCE (Proof Key for Code Exchange)](#pkce-proof-key-for-code-exchange)
- `redirect_uri` = `<callback_url>` — must match the original
- `code_verifier` = `<pkce_value>` — only if PKCE was used in Step 1

Successful response (HTTP 200):

```json
{
	"access_token": "<access_token>",
	"token_type": "Bearer",
	"expires_in": 1800,
	"refresh_token": "<refresh_token>",
	"scope": "gam_user_data",
	"user_guid": "139f4332-3f40-47b0-8fb4-ee7b3dbddc4f"
}
```

Notes:
- `refresh_token` is only returned if `Maximum OAuth token renewals` in GAM Security Policies was modified from its default; otherwise it is omitted
- `expires_in` defaults to 1800 seconds (30 minutes), configurable in the IDP Security Policies

GeneXus — exchange code for token:

```genexus
// In the callback Web Panel (receives ?code=…&state=…)
&Code = &HttpRequest.GetVariable(!"code")
&ReceivedState = &HttpRequest.GetVariable(!"state")
&ErrorCode = &HttpRequest.GetVariable(!"error_code")

&OriginalState = &WebSession.Get(!"OAuthState")
If &ReceivedState <> &OriginalState
	Msg(!"Error: State mismatch")
	Return
EndIf

If not &ErrorCode.IsEmpty()
	&ErrorMsg = &HttpRequest.GetVariable(!"error_message")
	Msg(!"IDP Error: " + &ErrorCode + !" - " + &ErrorMsg)
	Return
EndIf

&HttpClient = new()
&HttpClient.Host = &IDPHost
&HttpClient.BaseUrl = !"oauth/gam"
&HttpClient.Secure = 1
&HttpClient.AddVariable(!"grant_type", !"authorization_code")
&HttpClient.AddVariable(!"code", &Code)
&HttpClient.AddVariable(!"client_id", &ClientId)
&HttpClient.AddVariable(!"client_secret", &ClientSecret)
&HttpClient.AddVariable(!"redirect_uri", &CallbackURL)
&HttpClient.Execute(!"POST", !"access_token")

If &HttpClient.StatusCode = 200
	&TokenResponseJson = &HttpClient.ToString()
	&Oauth20AccessTokenSDT.FromJson(&TokenResponseJson)
	&WebSession.Set(!"GAMToken", &Oauth20AccessTokenSDT.ToJson())
Else
	Msg(!"Token exchange failed: " + &HttpClient.StatusCode.ToString().Trim())
	Return
EndIf
```

## Step 3 — UserInfo
Endpoint: `GET https://<idp_domain>/<virtual_dir>/oauth/gam/userinfo`

Headers:
- `Content-Type: application/x-www-form-urlencoded`
- `Authorization: <access_token>`

Successful response (HTTP 200):

```json
{
	"guid": "139f4332-3f40-47b0-8fb4-ee7b3dbddc4f",
	"username": "user",
	"email": "user@example.com",
	"verified_email": true,
	"first_name": "user",
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
	"CustomInfo": ""
}
```

GeneXus — get UserInfo:

```genexus
&StoredToken = &WebSession.Get(!"GAMToken")
&Oauth20AccessTokenSDT.FromJson(&StoredToken)

&HttpClient = new()
&HttpClient.Host = &IDPHost
&HttpClient.BaseUrl = !"oauth/gam"
&HttpClient.Secure = 1
&HttpClient.AddHeader(!"Authorization", &Oauth20AccessTokenSDT.access_token)
&HttpClient.Execute(!"GET", !"userinfo")

If &HttpClient.StatusCode = 200
	&UserInfoJson = &HttpClient.ToString()
	&UserInfoSDT.FromJson(&UserInfoJson)
	&WebSession.Set(!"GAMUser", &UserInfoJson)
Else
	Msg(!"UserInfo failed: " + &HttpClient.StatusCode.ToString().Trim())
EndIf
```

## Step 4 — Refresh Token
Endpoint: `POST https://<idp_domain>/<virtual_dir>/oauth/gam/access_token`

Use when the access_token expires. GAM returns HTTP 401 with error code `103` ("Token expired, log in again") when the token has expired

Body parameters:
- `grant_type` = `refresh_token`
- `client_id` = `<Client_ID>`
- `client_secret` = `<Client_Secret>`
- `refresh_token` = `<refresh_token_from_step2>`

Successful response (HTTP 200):

```json
{
	"access_token": "<access_token>",
	"token_type": "Bearer",
	"expires_in": 180,
	"refresh_token": "<refresh_token>",
	"scope": "gam_user_data",
	"user_guid": "139f4332-3f40-47b0-8fb4-ee7b3dbddc4f"
}
```

When to refresh:
- When a REST service returns HTTP 401 with `{"error": {"code": "103", "message": "Token expired, log in again."}}`
- If no refresh_token is available, restart the complete flow from Step 1

GeneXus — refresh token:

```genexus
&StoredToken = &WebSession.Get(!"GAMToken")
&Oauth20AccessTokenSDT.FromJson(&StoredToken)

If &Oauth20AccessTokenSDT.refresh_token.IsEmpty()
	Return
EndIf

&HttpClient = new()
&HttpClient.Host = &IDPHost
&HttpClient.BaseUrl = !"oauth/gam"
&HttpClient.Secure = 1
&HttpClient.AddVariable(!"grant_type", !"refresh_token")
&HttpClient.AddVariable(!"client_id", &ClientId)
&HttpClient.AddVariable(!"client_secret", &ClientSecret)
&HttpClient.AddVariable(!"refresh_token", &Oauth20AccessTokenSDT.refresh_token)
&HttpClient.Execute(!"POST", !"access_token")

If &HttpClient.StatusCode = 200
	&Oauth20AccessTokenSDT.FromJson(&HttpClient.ToString())
	&WebSession.Set(!"GAMToken", &Oauth20AccessTokenSDT.ToJson())
EndIf
```

## PKCE (Proof Key for Code Exchange)
PKCE adds security to the Authorization Code flow, especially for public clients without a secure client_secret

Configuration in IDP Backoffice:
- `OAuth 2.0 Option` = `Basic and PKCE` (compatible with non-PKCE clients) or `PKCE` (PKCE only)
- `Allowed PKCE Method` = `S256` or `plain`

Configuration via code:

```genexus
&GAMApplication.ClientAllowRemoteAuthenticationOAuth20Option = GAMOAuth20Options.BasicPKCE
// or GAMOAuth20Options.PKCE to enforce mandatory PKCE
```

PKCE flow:
- Step 1 (signin) — add `code_challenge_method` and `code_challenge` to the URL
- Step 3 (access_token) — add `code_verifier` to the POST body

`client_secret` is NOT required when PKCE is used: `client_id` + `code_verifier` are enough, on `authorization_code` and on `refresh_token`. This is what makes public clients (SPA, mobile) viable — they cannot hold a secret

```genexus
&CodeVerifier = GAMGenerateRandomString(64)
&WebSession.Set(!"PKCEVerifier", &CodeVerifier)

// code_challenge = SHA256(code_verifier) in base64url (for S256)
// code_challenge = code_verifier                      (for PLAIN)
&CodeChallenge = ComputeSHA256Hash(&CodeVerifier)

&SigninURL += !"&code_challenge_method=S256"
&SigninURL += !"&code_challenge=" + &CodeChallenge

&HttpClient.AddVariable(!"code_verifier", &WebSession.Get(!"PKCEVerifier"))
```

## Endpoints Summary
- Step 1 — `/oauth/gam/signin` (GET, redirect) — redirect user to IDP login
- Step 2 — `/oauth/gam/access_token` (POST) — exchange `code` for `access_token`
- Step 3 — `/oauth/gam/userinfo` (GET) — get authenticated user data
- Step 4 — `/oauth/gam/access_token` (POST) — refresh token when access_token expires

## Custom Callback URL
If the application uses a callback different from the default `/oauth/callback`:
- Configure the custom URL in `Callback URLs`
- Check `Custom callback URL?`

## Error Codes — Web IDP Flow
- HTTP redirect with `error_code` / `error_message` — Step 1 signin failed; inspect QueryString
- HTTP 401, code `103`, message `Token expired, log in again.` — use refresh_token (Step 4) or restart
- HTTP 400/401, code `114`, message `Invalid token` — Step 2 code invalid or expired; restart from Step 1

Full codes in `../error-codes-auth.md`

## Token Storage (Non-GAM Clients)
- `GAMToken` (WebSession key) — JSON of the access_token response (includes refresh_token)
- `GAMUser` (WebSession key) — JSON of the UserInfo response

CRITICAL: Without GAM, there is no `GAMSession` in DB. The session exists ONLY in the web server's `WebSession`. If the web server restarts, all sessions are lost

## Logout (Non-GAM Clients)
Pattern A — Redirect-based (recommended for SLO):

```genexus
&URL = &IDPURL.Trim() + !"/oauth/gam/signout"
&URL += !"?client_id=" + &ClientId
&URL += !"&redirect_uri=" + UrlEncode(&AfterLogoutURL)
&URL += !"&token=" + &Oauth20AccessTokenSDT.access_token
Link(&URL)
```

Pattern B — REST-based (no redirect):

```genexus
&HttpClient.AddHeader(!"Authorization", &Oauth20AccessTokenSDT.access_token)
&HttpClient.Execute(!"POST", !"signout")
&WebSession.Set(!"GAMToken", !"")
&WebSession.Set(!"GAMUser", !"")
```

## SLO Handler (Non-GAM Clients)
When the non-GAM client participates in an SLO chain, the IDP notifies the client to close its session:

- An SLO Web Panel must exist, accessible by URL (for example `{AppURL}/slo`)
- URL registered in `Custom Single Logout URLs` of the application at the IDP (API variable `ClientSingleLogoutCustomURLsSLO`)
- Receives via QueryString — `client_id`, `redirect_uri`, `state`, `token`
- Must invalidate session — compare `token` with the stored access_token, clean if match
- MUST auto-redirect to `redirect_uri?state=<state>`. Do NOT wait for user interaction

```genexus
// Web Panel: SLOHandler (URL: <your-app>/slo)
// Parm(in:&client_id, in:&redirect_uri, in:&token, in:&state)

Event Start
	&StoredToken = &WebSession.Get(!"GAMToken")
	&Oauth20AccessTokenSDT.FromJson(&StoredToken)
	If &Oauth20AccessTokenSDT.access_token = &token.Trim()
		&WebSession.Set(!"GAMToken", !"")
		&WebSession.Set(!"GAMUser", !"")
	EndIf
	&ReturnURL = Format(!"%1?state=%2", &redirect_uri, &state)
	Link(&ReturnURL)
Endevent
```

Common handler errors:
- Format `?&state=` (extra ampersand) instead of `?state=`
- Requires button click for redirect — breaks the automatic chain
- Does not validate the token before cleaning the session — could clean another user's session

Backoffice location of the SLO URL: `Applications → Edit → OAuth Authentication tab → WEB section → Custom Single Logout URLs`

## Differences with GAM Clients
- Primary flow — GAM client uses GAMRemote automatically; non-GAM client uses HttpClient + redirect manually
- Session storage — GAM client uses `GAMSession` in DB; non-GAM client uses web server `WebSession`
- Token refresh — GAM client refreshes automatically; non-GAM client refreshes manually or re-authenticates
- SLO callback URL — GAM client uses `oauth/gam/callback` auto-built; non-GAM client registers a custom URL
- GAM traces — GAM client emits traces if tracing is enabled; non-GAM client has no GAM module, traces live on the IDP
- Auto-provisioning — handled by GAM on GAM clients; not applicable for non-GAM clients (user must exist on the IDP)
- PKCE — GAM client configured via auth type; non-GAM client implements PKCE manually
