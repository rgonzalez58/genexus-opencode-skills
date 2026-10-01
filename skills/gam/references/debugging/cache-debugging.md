---
name: cache-debugging
description: GAM cache behavior — how to detect stale cache and force invalidation
---

# GAM Cache Debugging
GAM caches repository data (roles, permissions, security policies, authentication types) and session data. When configuration changes don't take effect, stale cache is the most common cause

Related files:
- [GAM Debugging Core](common-debugging.md) — trace activation (cache also affects trace switches)
- [GAM SQL Diagnostic Queries](sql-diagnostic-queries.md) — cache timeout queries
- [GAM Multi-Tenant & Repository Encyclopedia](../multi-tenant/domain-multitenant.md) — repository cache model

---

## Symptoms of Stale Cache
- Permission changes not reflected — likely stale Role-Permission cache. Fix with `GAMRepository.ClearCache()`
- Security policy changes ignored — likely stale General settings cache. Fix with `GAMRepository.ClearCache()`
- New authentication type not available — likely stale Auth type cache. Fix with `GAMRepository.ClearCache()`
- User sees old role after assignment change — likely stale User session cache. Wait for `UserSessionCacheTimeout` or re-login
- Backoffice changes visible but app behaves as before — likely stale App-level cache. Fix with `GAMRepository.ClearCache()` + restart AppPool

## Cache Configuration
- `CacheTimeout` in `GAMRepository.CacheTimeout`
	* Purpose: duration in seconds for repository-level cache
	* Default: varies by version
- `UserSessionCacheTimeout` in `GAMRepository.UserSessionCacheTimeout`
	* Purpose: duration in seconds for session-level cache
	* Default: varies by version

## Force Cache Invalidation
```genexus
GAMRepository.ClearCache()
```

For immediate invalidation across all cached keys, also recycle the AppPool (IIS) or restart Kestrel / Tomcat

## Diagnostic SQL — Cache Settings
```sql
SELECT RepPropId, RepPropVal
FROM gam.RepositoryProp
WHERE RepPropId IN ('CacheTimeout', 'UserSessionCacheTimeout')
  AND RepPropRepId = '<REPOSITORY_GUID>';
```

## Trace Signature — Cache
```
GAMTrace-GAMGetGAMGeneralSetting - GetCache name:com.genexus.gam.general
```

This trace confirms cached general settings are being loaded. If the `Value` JSON shows unexpected values, cache may be stale

## Cache Hierarchy
- Repository-level cache — settings, roles, permissions, auth types. Key: `com.genexus.gam.repositories`
- Session-level cache — current user session data. Key per session GUID
- General settings cache — `isGAMMultiTenant`, version, global flags. Key: `com.genexus.gam.general`

Each level has its own timeout. Invalidate the one whose data changed

## OAuth Flow Cache Keys (v18u15+)
Format B cache traces surface as `genexus.security.api.GAMGetCache - Cache-NotFound` / `Cache-Found` and `genexus.security.api.GAMSetCache - End_Method`. The `CacheName` and `Key` JSON fields identify which slot was hit. Canonical flow: [GAM-as-SP — OAuth 2.0 Common Flow (observable in trace)](../authentication/external-providers/oauth20/common/oauth20-common-flow.md)

Cache slots observed across OAuth 2.0 SIGNIN and SLO:

- `com.genexus.gam.connectionfile` — keyed by connection name. DB connection block, populated on page load
- `com.genexus.gam.repositories` — keyed by `<RepositoryGUID>`. Full repository metadata
- `com.genexus.gam.applicationfile` — keyed by `<AppName>`. Application block
- `com.genexus.gam.applications.repository_<RepoGUID>` — sub-keys `AppGUID:<GUID>-` and `AppCliId:<ClientID>-`
- `com.genexus.gam.general` — key `GeneralSettings` carries the multi-tenant flag, tracing flag, OIDC issuer and `KeysExpirationTime`. Key `LastRunDemon` carries state-cache cleanup bookkeeping
- `com.genexus.gam.eventsubs.repository_<RepoGUID>` — keyed by `Event:<eventName>-`, for example `Event:repository-login-`, `Event:user-insert-`, `Event:repository-logout-`. `Cache-NotFound` here is benign: no subscribers registered
- `com.genexus.gam.apppermission.repository_<RepoGUID>` — keyed by `AppId:<id>-Prm:<perm>`. Permission-check cache

## Stale-Config Gotcha During OAuth Flows
NONE of the caches listed in "OAuth Flow Cache Keys (v18u15+)" above are purged during an OAuth 2.0 SIGNIN or SLO. Configuration changes made through the Backoffice while a user is mid-session are NOT observed until one of the following happens:
- The cache TTL for that slot expires
- `GAMRepository.ClearCache()` is invoked from app code
- The application host is restarted (AppPool recycle on IIS, Kestrel restart, Tomcat restart)

Forcing a refresh — ordered by impact:
- Targeted invalidation from app code: `GAMRepository.ClearCache()` (see "Force Cache Invalidation" above)
- AppPool recycle / Kestrel restart / Tomcat restart for full eviction across all slots
- Wait for TTL if neither is acceptable (TTLs configured via `CacheTimeout`, see "Cache Configuration" above)

Evidence pattern in a log: the same `Cache-Found` entry for the slot containing the just-changed configuration value, followed by downstream procedures still reading the old value. Compare the `Value` payload from `GAMSetCache` against what the Backoffice currently shows
