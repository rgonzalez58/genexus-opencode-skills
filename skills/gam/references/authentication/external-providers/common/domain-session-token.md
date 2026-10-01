---
name: domain-session-token
description: Token structure, session types, validation flow, expiration lifecycle, session audit, KillSession
---

# GAM Session and Token Reference
Related files:
- [GAM Authorization — Permission Evaluation](../../../authorization/domain-authz.md) — permission evaluation using the validated session
- [GAM Security Policies Encyclopedia](../../../authorization/domain-security-policies.md) — OauthTokenExpire, WebSessionTimeout, policy-level controls
- [../../../debugging/trace-signatures-catalog.md#session](../../../debugging/trace-signatures-catalog.md#session-session) — session trace signatures

## Token Validation Flow (on each request)
On every request to a protected resource, GAM middleware performs token validation:

- Extract token from `Authorization: Bearer <token>` header or session cookie
- Look up active session for that token
- Verify application scope — the session must belong to the requesting Application. If mismatched, returns `AccessDenied` (token from a different app)
- Check expiration — if the session has expired, it is marked inactive and returns `SessionExpired` (Code 17)
- Sliding expiration — if valid, the session's last access time is updated (extends the timeout window)
- Result — if valid, the request proceeds with the user's identity injected into context. If invalid, the user is redirected to login or receives HTTP 401

If no matching active session is found, returns `InvalidToken` (Code 114)

---

## Session Management and Timeout Conflict
GAM has TWO expiration clocks running in parallel. If they are not aligned, the user experiences erratic behavior

- Web Server Session
	* Lives in: IIS/Tomcat (memory)
	* Configuration: `web.config` sessionTimeout / Tomcat `session-timeout`
- OauthTokenExpire
	* Lives in: GAMSecurityPolicy (DB)
	* Configuration: Backoffice → Security Policies → Token Expiration

Problem: if `OauthTokenExpire` (DB) < Server Session Timeout (memory):

- The UI appears logged in (server session cookie is still alive)
- But AJAX calls to the backend fail (GAM token expired in DB)
- The user sees intermittent errors without understanding why

Fix: `OauthTokenExpire` >= Web Server Session Timeout. Always

### Session Parameters in SecurityPolicy
- `OauthTokenExpire`
	* Default: 3600 (1h)
	* Effect: duration of the access_token in DB
- `WebSessionTimeout`
	* Default: 20 min
	* Effect: web server session timeout
- `MaxConcurrentSessions`
	* Default: 0 (unlimited)
	* Effect: maximum concurrent sessions per user
- `SlidingExpiration`
	* Default: True
	* Effect: if True, each request renews the timeout

### Session Hierarchy (Parent/Child)
```
Parent Session (IDP GAM)
  ├── Child Session (Client App 1)
  ├── Child Session (Client App 2)
  └── Child Session (Client App 3)
```

The Parent Session has `TokenToFinish` which controls SLO. When the parent is closed, all child sessions are invalidated. See [GAM Single Log Out (SLO)](../../../logout/domain-slo.md) for the SLO 3-phase flow

---

## Token Structure
GAM tokens follow the pattern: `<RepositoryGUID>!<HashString>`

Example: `<repository-guid>!<session-hash>`

- `<repository-guid>`
	* Repository GUID that identifies which repository owns this session
- `!`
	* Separator
- `<session-hash>`
	* Hashed session identifier, unique per session

## Session Hierarchy (IDP Scenario)
When a client authenticates via an IDP, TWO sessions are created at the IDP:

```
ParentToken (IDP session)         ← SesExtToken
  └── ChildToken (client session) ← SesExtToken2 / TokenToFinish
```

- ParentToken
	* The IDP's own session
	* Killed only if `GAMRemoteLogoutBehavior` = `cliip` or `clial`
- ChildToken
	* Represents the client at the IDP
	* Always killed during SLO

## Session Types
- Value `1`
	* Web Session

## Key Session Fields (visible in trace JSON dumps)
These fields appear in trace dumps during authentication flows and are useful for diagnosing state issues:

- `GAMTokenState`
	* The state key (with prefix indicating flow type)
- `RepositoryGUID`
	* Repository this session belongs to
- `ConnectionName`
	* Connection used for this session
- `ApplicationId`
	* Application ID within the repository
- `AuthenticationTypeName`
	* Auth type used (e.g., `gamremote`, `oauth20`, `local`)
- `SessionType`
	* Session type (1 = web)
- `FromURL`
	* Where to redirect after the flow completes
- `ErrorURL`
	* Where to redirect on error

## External Tokens (JWT)
When authenticating via OAuth 2.0 / OIDC, the IDP stores the external JWT:
```json
{"ExternalToken": "<external_jwt>"}
```
`<external_jwt>` is the raw JWT issued by the external IDP (starts with `eyJ`, the base64-encoded JOSE header). This JWT is used during SLO when `SLOEnable=true` — GAM reads `ExternalToken` from the session to build the logout request to the external IDP

## Two-Factor Authentication Fields
```json
{
  "TwoFactorAuthentication": {
	"Enable": false,
	"AuthenticationType": "",
	"AuthenticationTypeName": "",
	"FirstAuthenticationFactorExpiration": 0,
	"ForceForAllUsers": false,
	"IsOTP": false,
	"IsTOTP": false
  },
  "UseTwoFactorAuthentication": false
}
```

## Session Finish Traces
```
GAMTrace-SLOProcess - IDP - Finish token:<CHILD_TOKEN>
GAMTrace-SLOProcess - IDP - Finish parent token:<PARENT_TOKEN>
```

## State Persistence Across Redirects
GAM persists temporary state across redirects for cross-redirect continuity:
- The client-side handler writes, reads, and deletes entries by state key
- The IDP-side handler writes, reads, and deletes entries by state key
- Entries auto-expire (a background cleanup process removes entries older than approximately 1 hour)
- Trace for deletion: `GAMTrace-GAMStateClientAPI - Delete State:<KEY>`

### CRITICAL: `IDP-` Prefix Mechanism (Shared-DB Collision Prevention)
When IDP and Client share the same GAM database, both the client-side and IDP-side state handlers process the SAME OAuth `state` parameter from the URL

The `state` string is hashed to generate the lookup key for persisted state

The fix (present in u12/u14HF, accidentally removed in u13HF/u14): The IDP-side handler prefixes the raw `state` string with `IDP-` BEFORE hashing it. This produces a completely different hash, preventing collision

The two persisted state entries for the same flow (Correct Behavior):

- Client state
	* Raw string (before hashing): `GRESTDl2…`
	* Written by: Client-side handler
- IDP state
	* Raw string (before hashing): `IDP-GRESTDl2…`
	* Written by: IDP-side handler

Without the `IDP-` prefix:
Both sides hash the same raw string, producing the same lookup key. The second write causes a PRIMARY KEY violation, resulting in `error_code=532 "El proceso de autenticacion fallo"`

### Version Regression
- u12
	* `IDP-` Prefix: Present
	* Status: Works
- u13 Release
	* `IDP-` Prefix: Present
	* Status: Works
- u13 HotFix
	* `IDP-` Prefix: Removed
	* Status: BUG
- u14
	* `IDP-` Prefix: Removed
	* Status: BUG
- u14 HotFix
	* `IDP-` Prefix: Fixed
	* Status: Works

### When does this NOT fail?
- IDP and Client use SEPARATE databases — different state storage — no collision
- This is why APP1 might work and APP2 fails: depends on DB configuration per app

---

## Session Audit and Management
GAM provides programmatic access to session logs for auditing, monitoring, and security incident response through `GAMSessionLog` and `GAMRepository` methods

### `GAMSessionLog` Properties
- `Token`
	* Session token identifier
- `User`
	* User associated to the session
- `LoginDate`
	* Session login date-time
- `LogoutDate`
	* Session logout date-time
- `IsAlive`
	* Whether the session is currently active
- `LoginRetries`
	* Number of failed login attempts for this session
- `LoginRetryCount`
	* Configured retry threshold (from Security Policy)
- `FullLog`
	* Whether extended session audit is enabled

### Querying Session Logs
```genexus
// Get session logs with filter
&SessionLogs = GAMRepository.GetSessionLogs(&Filter, &Errors)

// Get session logs with filter and sort order
&SessionLogs = GAMRepository.GetSessionLogsOrderBy(&Filter, GAMSessionLogListOrder.Date_Desc, &Errors)

// Count total sessions matching a filter
&Count = GAMRepository.GetSessionLogsCount(&Filter)

// Count currently active sessions
&AliveCount = GAMRepository.GetAliveSessionCount(&Errors)
```

### Killing Sessions (`KillSession`)
Use `KillSession` to invalidate a session — typically for security incidents (compromised tokens, suspicious activity) or administrative session management

```genexus
// Kill a specific session by token
&Ok = GAMSessionLog.KillSession(&CompromisedToken, &Errors)
If not &Ok
		msg(!"Unable to close session")
EndIf
```

### Expired Session Cleanup
```genexus
// Update status of expired session logs (maintenance task)
&Ok = GAM.UpdateExpiredSessionLog(&ProcessFilter, &Errors)
```

### Constraints — Session Audit
- `KillSession` requires the caller to have permissions to manage repository sessions
- Session log queries are executed through `GAMRepository`, not `GAMSessionLog` directly
- `FullLog` impacts audit verbosity and storage volume — enable selectively
- `GetAliveSessionCount` requires GAM Manager Repository connection
- After killing a session, the user's next request will receive `InvalidToken` (Code 114) and must re-authenticate
