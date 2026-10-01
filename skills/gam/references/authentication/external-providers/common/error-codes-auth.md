---
name: error-codes-auth
description: Complete catalog of GAM authentication error codes — Local Login, OAuth 2.0, OIDC, Web IDP, REST IDP, auto-provisioning
---

# GAM Authentication — Error Code Catalog
Related files:
- [GAM Authentication](domain-auth.md) — Authentication flow signatures and Silent Skip detection
- [GAM as Web IDP Server — Authorization Code Flow](../../local-idp/authorization-code-flow.md) — Web IDP endpoint responses (Authorization Code)
- [GAM as REST IDP Server — Password Grant Flow](../../local-idp/password-grant-flow.md) — REST IDP endpoint responses (Password Grant)
- [GAM Debugging Core](../../../debugging/common-debugging.md) — How to activate traces for error diagnosis

## Code 10 (InvalidCredentials) — Local Login
- Cause: wrong password or user does not exist
- Associated trace: `GAMTrace-GAMAuthenticationLogin` without "Session created"
- Fix: verify credentials. GAM does NOT distinguish "user not found" from "wrong password" by design

## Code 11 (InvalidPassword) — Local Login
- Cause: password does not meet format requirements
- Associated trace: before hash comparison
- Fix: verify it meets the password policy

## Code 12 (UserNotActive) — Local Login
- Cause: user is inactive
- Associated trace: after "User found"
- Fix: activate user in Backoffice

## Code 17 (UserLocked) — Local Login
- Cause: exceeded `MaxFailedLoginAttempts`
- Associated trace: "User LOCKED"
- Fix: unlock in Backoffice or wait for `AccountLockDuration`

## Code 22 (PasswordMustChange) — Local Login
- Cause: password expired per policy
- Associated trace: after validation OK
- Fix: force password change

## Code 30 (ApplicationNotFound) — All modules
- Cause: AppId/ClientId does not exist
- Associated trace: early in flow
- Fix: verify the application is registered and the `ClientID` in the request matches

## Code 103 (TokenExpired) — Web IDP / REST
- Cause: token expired
- Associated trace: HTTP 401, "Token expired, log in again"
- Fix: use refresh_token flow or restart authentication

## Code 114 (InvalidToken) — OAuth 2.0 / OIDC
- Cause: token exchange failed or Silent Skip
- Associated trace: ABSENCE of `AddHeader: Authorization`
- Fix: verify `token_type` is not empty in IDP response, verify `access_token` is valid

## Code 114 (InvalidToken) — Callback without state match
- Cause: invalid state (not found in `LoginTmp`)
- Associated trace: no `GAMTrace-ValidStateInDB OK`
- Fix: state expired (`GAMLoginTmpDemon` cleaned it) or CSRF. Restart flow from Step 1

## Code 515 (OIDCValidationFailed) — OIDC
- Cause: invalid JWT signature / claims
- Associated trace: `GAMTrace-OIDC-Error when on JWT verification`
- Fix: verify JWKS endpoint, key rotation, clock skew. For `iss` / `aud` mismatch, check the configuration in GAM Backoffice

## Code 532 (UserNotFound) — Auto-provisioning
- Cause: GUID does not exist in GAM
- Associated trace: after UserInfo response
- Fix: verify auto-provisioning is enabled for this auth type, or create the user manually

## Code 621 (InvalidURLAfterSLO) — Single Log Out
- Cause: the `redirect_uri` sent to `/oauth/gam/signout` is not in the Application's `ClientSingleLogoutValidURLsAfterSLO` list
- Associated trace: `GAMValidURLFromAList - End_Method` returns the 621 error. The preceding `Start_Method` exposes both sides of the comparison — `Parm2` is the configured list, `Parm3` is the URL being validated. `GAMRemoteLogout - SLOProcess-AfterSLO` logs the effective list value read from config
- Fix: make the two values match. The comparison is LITERAL — a trailing slash is significant, so `http://localhost:5173/` does NOT match a list entry of `http://localhost:5173`. Either align the client's `redirect_uri` with the configured entry, or register both variants in the list (comma-separated)
- Note: when the trace shows the current Backoffice value in `SLOProcess-AfterSLO`, config cache is NOT the cause — compare `Parm2` against `Parm3` character by character before suspecting stale cache
