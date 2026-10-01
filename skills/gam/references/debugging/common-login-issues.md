---
name: common-login-issues
description: Common GAM login failure patterns and their root causes
---

# Common GAM Login Issues
Scope: quick reference for recurring login failure patterns

Related files:
- [GAM Debugging Core](common-debugging.md) — trace activation and diagnostic workflow
- [GAM SQL Diagnostic Queries](sql-diagnostic-queries.md) — user state and repository queries
- [GAM Authentication](../authentication/external-providers/common/domain-auth.md) — authentication flows

---

## "Page Refreshes on Login" (no error visible)
Root cause: HTTP 440 (AJAX token mismatch) causing `gxgral.js` to reload the page

Diagnosis:
- Open browser DevTools Network tab
- Submit login
- Look for POST to login endpoint returning 440
- If 440: the session expired or `AJAX_SECURITY_TOKEN` was invalidated

Common causes:
- IIS AppPool recycled between page load and form submit
- Session timeout too short
- Multiple IIS worker processes (web garden) causing session affinity issues

## No Redirect After Login
Root cause: `gamhome` object exists but no Home Object is configured for the GAM Application

Fix: in GAM Backoffice, set the Home Object for the Application, or configure `GAMApplication.HomeObject` programmatically

## Admin User Locked or Inactive
Diagnosis query (see [GAM SQL Diagnostic Queries](sql-diagnostic-queries.md)):

```sql
SELECT UserGUID, UserName, UserEMail, UserIsBlk, UserIsAct, UserIsDlt
FROM gam.[User]
WHERE UserName = 'admin';
```

- `UserIsBlk` — user is blocked. Expected `0`
- `UserIsAct` — user is activated. Expected `1`
- `UserIsDlt` — user is deleted. Expected `0`

Fix if locked:

```sql
UPDATE gam.[User] SET UserIsBlk = 0 WHERE UserName = 'admin';
```

## Silent Skip (external IDP not invoked)
Root cause: GAM detected an already-valid session and skipped the external authentication step. Often happens after testing when the previous session is still active

Diagnosis: check [Silent Skip Anti-Pattern](../authentication/external-providers/common/domain-auth.md#silent-skip-anti-pattern--complete-reference)

## Login Loop — back to Login after Success
Common causes:
- `redirect_uri` misconfigured (returns to login)
- Role without Home Object and no default redirect
- Token validation rejects the just-created session (Application scope mismatch)

Diagnosis: follow the trace sequence from Login success → Session created → Token validation result

## Duplicate Key Violation on `gam.LoginTmp`
Root cause: shared-DB collision when IDP and client share the same GAM DB without the `IDP-` prefix. See [GAM Session and Token Reference](../authentication/external-providers/common/domain-session-token.md) "Shared-DB Collision" section

Fix: upgrade to a GAM version with the `IDP-` prefix (u12, u13 Release, u14 HotFix+) or use separate databases

## OAuth 2.0 External-Login Silent Failures (v18u15+)
Catalog of silent-failure conditions for OAuth 2.0 external login (GAM-as-SP against an external IDP). Trace signatures follow Format B (`genexus.security.api.*` structured JSON). Canonical flow definition: [GAM-as-SP — OAuth 2.0 Common Flow (observable in trace)](../authentication/external-providers/oauth20/common/oauth20-common-flow.md). Signatures catalogued in [GAM Trace Signatures Catalog](trace-signatures-catalog.md) Format B section

### State missing on callback
- Condition: IDP strips the `state` query parameter on return
- Signature: `GAMExternalAuthenticationOAuth20 - GoToIP-AuthType-OAuth20 - {"data":{"URL":"…&state=<val>…"}}` present on the outbound, but no `GAMStateClientAPI - Start_Method - {"data":{"Parm1":"gem"}}` on the callback
- Root cause: IDP policy does not round-trip `state`
- Fix: verify IDP configuration and `Authorize.State_Include = True` in the GAM auth type

### State expired
- Condition: user paused at IDP consent longer than the state TTL
- Signature: `GAMStateClientAPI - Start_Method - {"data":{"Parm1":"gem"}}` returns `{"retval":{"GAMTokenState":""}}`
- Root cause: state record removed by cleanup demon before the callback returned
- Fix: shorten the user-facing IDP timeout or increase the state TTL

### State double-consumed
- Condition: browser Back / Forward, or a second tab replaying the callback URL
- Signature: second callback finds empty `GAMTokenState` (same as 7.2) although the first callback succeeded
- Root cause: `Parm1 = "gem"` removes on read, so the second call finds nothing
- Fix: educate users, or add idempotent handling at the app layer

### Token exchange fails with `invalid_grant`
- Condition: IDP requires PKCE but PKCE is not enabled in the GAM auth type
- Signature: `GAMSearchJsonLabel - End_Method - {"data":{"Label":"error","retval":"invalid_grant"}}` during `ReturnFromIP-*`
- Root cause: missing `code_verifier` in the token request
- Fix: set `PKCEAuthentication.Enable = True` with `Method = "S256"`

### Token exchange fails with `invalid_client`
- Condition: credentials transport mismatch (Basic header vs body-form)
- Signature: `GAMSearchJsonLabel - End_Method - {"data":{"Label":"error","retval":"invalid_client"}}` after `ReturnFromIP-Add-Header-AuthType-OAuth20`
- Root cause: `Header_AuthorizationBasic_Include` does not match the IDP's expected transport
- Fix: toggle Basic header vs body-form credentials per IDP docs

### UserInfo claim-mapping failure — every login creates a new user
- Condition: the mapped external-id claim does not match the IDP's canonical id claim
- Signature: `GAMSearchJsonLabel - End_Method - {"data":{"Label":"<mapped>","retval":""}}` with empty `retval`, followed by `GAMUpdateOrCreateUserInGAM` inserting a new row on every login
- Root cause: `ResponseUserExternalId_Name` set to a claim the IDP does not emit (for example `id` when the IDP returns `sub`)
- Fix: confirm the IDP's canonical id claim and update `ResponseUserExternalId_Name` accordingly

### First login OK, second login shows "session expired"
- Condition: synchronous event listener invalidates the fresh session, or `OauthTokenExpire` is shorter than the web-server session timeout
- Signature: `ExecuteEventSubscriptions - Start_Sub-ExecuteExternalObject` emits an error, or `OauthTokenExpire` configured below the web server session timeout
- Root cause: listener aborts session setup, or token-expiration timeout conflict
- Fix: inspect the listener for exceptions. See [Error Codes](../authorization/domain-authz.md#error-codes) for the timeout-conflict pattern

### Post-login URL malformed with `%3A%2F%2F` in the browser bar
- Condition: Java generator in historical versions double-encodes `FromURL`
- Signature: `ChangeURLToCurrentVirtualDir - End_Method - {"data":{"retval":"http%3A%2F%2F…"}}`
- Root cause: Java-generator `FromURL` URL-encoding bug
- Fix: upgrade to a fixed GeneXus / GAM version or apply the known KB workaround

### No authorize redirect — login page stays
- Condition: the configured external auth type is disabled or not set to auto-redirect
- Signature: no `GAMExternalAuthenticationOAuth20 - Start_Method - {"data":{"Parm1":1}}` line at all after `GAMAuthenticationLogin - Start_Method`
- Root cause: `AuthenticationOAuth20SDT.IsEnable = False` or `RedirectToAuthenticate = False`
- Fix: verify the auth type flags in the Backoffice
