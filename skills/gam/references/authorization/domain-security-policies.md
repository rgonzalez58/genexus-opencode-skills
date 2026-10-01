---
name: domain-security-policies
description: Password policies, account locking, session timeout, cache management, biometrics, token reuse
---

# GAM Security Policies Encyclopedia
## Overview
Security Policies in GAM control password rules, account locking, session behavior, and logout validation. They are configured per-repository via the GAM Web Backoffice (Settings → Security Policies)

## Password Policies
- Minimum length
	* Minimum characters required
- Require uppercase
	* Must contain at least one uppercase letter
- Require lowercase
	* Must contain at least one lowercase letter
- Require numbers
	* Must contain at least one digit
- Require special characters
	* Must contain at least one special character
- Password expiration (days)
	* Days before password must be changed
- Password history
	* Number of previous passwords that cannot be reused

## Account Locking
- Max failed attempts
	* Number of failed logins before account locks
- Lock duration (minutes)
	* Time the account stays locked
- Auto-unlock
	* Whether locked accounts automatically unlock after duration

## Session Policies
- Session timeout (minutes)
	* Idle time before session expires
- Max concurrent sessions
	* Maximum simultaneous sessions per user
- `RememberUser`
	* Enables "Remember Me" for SSO persistence

## SLO-Related Settings
- `GAMRemoteLogoutBehavior`
	* Controls session kill scope during SLO
	* Values: `clionl` (client only) / `cliip` (client+IDP) / `clial` (all)
- `SLOEnable`
	* Enables SLO for an authentication type
	* Values: `true` / `false`
- `ClientSingleLogoutValidURLsAfterSLO`
	* Whitelist of valid redirect URLs after SLO
	* Values: Comma-separated URLs
- `UseAbsoluteUrlByEnvironment`
	* Use absolute URLs based on environment config
	* Values: `true` / `false`

## Trace Signatures
When security policies are loaded, GAM emits a trace showing the cached general settings:
```
GAMTrace-GAMGetGAMGeneralSetting - GetCache name:com.genexus.gam.general
	Key:GeneralSettings:   Value:{"EnableTracing":1,"isGAMMultiTenant":false,"Version":"4.1.5",…}
```
This trace confirms: tracing is active (`EnableTracing:1`), multi-tenant status, and GAM version. Useful to verify that the correct security policy configuration is being loaded from cache

## Repository-Level Security Properties (`GAMRepository`)
These properties are configured at the Repository level and affect authentication behavior:

- `CacheTimeout`
	* Type: Numeric (seconds)
	* Cache duration for repository-level data. Cached data (roles, permissions, settings) is served until expiration. Use `GAMRepository.ClearCache()` after changes that must take effect immediately
- `UserSessionCacheTimeout`
	* Type: Numeric (seconds)
	* Cache duration for user session data. Shorter values = more DB queries but fresher session state
- `GAMUnblockUserTimeout`
	* Type: Numeric (minutes)
	* Minutes before a locked user is automatically unblocked after exceeding max login retries
- `EnableReusingActiveUserTokens`
	* Type: Boolean
	* When `True`, if a user authenticates again while an active token exists, GAM reuses the existing token instead of creating a new one. Reduces token proliferation in high-frequency login scenarios
- `ConnectionChallengeExpire`
	* Type: Numeric (minutes)
	* Expiration timeout for connection security challenge handshakes. Applies to challenge-response authentication flows
- `EnableBiometrics`
	* Type: Boolean
	* Enables biometric authentication support (fingerprint, face recognition) where the platform supports it (Smart Devices)

### Cache Management
When configuration changes (roles, permissions, security policies) don't take effect immediately, it's typically because they're cached:

```genexus
// Force cache invalidation after administrative changes
GAMRepository.ClearCache()
```

When to clear cache:
- After modifying roles or permissions in Backoffice (if changes aren't reflected)
- After updating Security Policy settings via code
- After modifying Authentication Types
- During debugging when stale data is suspected

## Password Policy Properties via Code (`GAMSecurityPolicy`)
Security policies can also be managed programmatically through the `GAMSecurityPolicy` External Object:

- `AllowMultipleConcurrentWebSessions`
	* Whether a user can have multiple simultaneous web sessions
- `WebSessionTimeout`
	* Web session timeout value
- `OauthTokenExpire`
	* OAuth token expiration value
- `OauthTokenMaximumRenovations`
	* Maximum OAuth token renewals
- `MinimumTimeToChangePasswords`
	* Minimum elapsed time between password changes
- `MaximumPasswordHistoryEntries`
	* Previous passwords that cannot be reused
- `MinimumNumericCharactersPassword`
	* Required minimum numeric characters in password
- `MinimumUpperCaseCharactersPassword`
	* Required minimum uppercase characters in password
- `MinimumSpecialCharactersPassword`
	* Required minimum special characters in password

Hardening a policy means loading it and setting any subset of the properties listed above (e.g., `AllowMultipleConcurrentWebSessions = False`, tighter `Minimum*` password properties) — same idempotent Load/Save/error shape as [Idempotent Save Pattern](../debugging/common-genexus-patterns.md#idempotent-save-pattern-load-or-new)

Constraints for SecurityPolicy via Code:
- After `Load()` and `Save()`, check `Success()` / `Fail()`
- Changes via code are equivalent to changes via Backoffice — they update the same database records
- Cache may delay visibility of changes — call `GAMRepository.ClearCache()` if needed

---

## Common Issues
- Session expires too fast: Check `Session timeout` in Security Policies AND `OauthTokenExpire` — see Timeout Conflict in [GAM Authorization — Permission Evaluation](./domain-authz.md)
- User locked out: Check `Max failed attempts` and `Lock duration`. Also check `GAMUnblockUserTimeout` for auto-unlock timing
- SLO redirect blocked: The redirect URL is not in `ClientSingleLogoutValidURLsAfterSLO` whitelist
- Password rejected: Doesn't meet the password policy rules — check all `Minimum*` properties
- Config changes not taking effect: Cache may be stale — call `GAMRepository.ClearCache()` or wait for `CacheTimeout` to expire
- Token reuse unexpected: Check `EnableReusingActiveUserTokens` — if `True`, repeated logins may return the same token
