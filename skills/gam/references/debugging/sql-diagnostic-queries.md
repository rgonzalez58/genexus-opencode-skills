---
name: sql-diagnostic-queries
description: Key SQL queries for diagnosing GAM state — users, repositories, applications, auth types, sessions
---

# GAM SQL Diagnostic Queries
Scope: direct database inspection queries for GAM diagnostics. Use when traces alone are not enough or to verify a trace-diagnosed state

Schema note: all GAM tables use the `gam` schema. The `[User]` table name requires square brackets (reserved word)

Column naming: GAM uses shortened column names

- `UserIsBlk` (user is blocked, not `UserIsBlocked`)
- `UserIsAct` (user is activated, not `UserIsActivated`)
- `UserIsDlt` (user is deleted, not `UserIsDeleted`)
- `RepPropVal` (not `RepositoryPropertyValue`)
- `SysParVal` (not `SystemParameterValue`)
- `SesSts` (not `SessionStatus`)
- `SesExtToken2` (not `SessionExternalToken2`)

Related files:
- [GAM Debugging Core](common-debugging.md) — trace activation (these queries verify the switches)
- [GAM Cache Debugging](cache-debugging.md) — cache timeout queries
- [GAM Multi-Tenant & Repository Encyclopedia](../multi-tenant/domain-multitenant.md) — repository model

---

## System Configuration
All system parameters (master switches):

```sql
SELECT SysParId, SysParVal FROM gam.SysPar;
```

Tracing status per repository:

```sql
SELECT RepId, RepPropId, RepPropVal
FROM gam.RepositoryProp
WHERE RepPropId = 'EnableTracing';
```

Cache timeout per repository:

```sql
SELECT RepId, RepPropId, RepPropVal
FROM gam.RepositoryProp
WHERE RepPropId = 'CacheTimeout';
```

## Users
All users with status flags:

```sql
SELECT UserGUID, UserName, UserEMail, UserIsBlk, UserIsAct, UserIsDlt
FROM gam.[User];
```

Find a specific user:

```sql
SELECT UserGUID, UserName, UserEMail, UserIsBlk, UserIsAct, UserIsDlt
FROM gam.[User]
WHERE UserName = 'admin' OR UserEMail = 'admin@example.com';
```

Unlock a user:

```sql
UPDATE gam.[User] SET UserIsBlk = 0 WHERE UserName = 'admin';
```

## Repositories and Applications
All repositories:

```sql
SELECT RepId, RepName, RepGUID, RepDefAutTypeName, RepUserIdentification
FROM gam.Repository;
```

All applications:

```sql
SELECT RepId, AppId, AppName, AppGUID, AppIsBaseApplication
FROM gam.Application;
```

Applications for a specific repository:

```sql
SELECT a.AppId, a.AppName, a.AppGUID, a.AppIsBaseApplication, r.RepName
FROM gam.Application a
JOIN gam.Repository r ON a.RepId = r.RepId;
```

## Authentication Types
All configured authentication types:

```sql
SELECT * FROM gam.AuthenticationType;
```

Authentication types for a specific repository:

```sql
SELECT at.*
FROM gam.AuthenticationType at
JOIN gam.Repository r ON at.RepId = r.RepId
WHERE r.RepName = 'MyRepository';
```

## Security Policies
All security policies:

```sql
SELECT * FROM gam.SecurityPolicy;
```

## Sessions
Recent sessions (useful for debugging login issues):

```sql
SELECT TOP 20 SesId, SesGUID, SesSts, SesStartDate, SesEndDate
FROM gam.Session
ORDER BY SesStartDate DESC;
```

Daughter sessions for a parent session (SLO diagnostics):

```sql
SELECT SesGUID, SesCliId, SesSts, SesEndDate, SesParGUID
FROM GAMSession
WHERE SesParGUID = '<parent_session_guid>'
  AND SesSts = 'F';
```

Interpretation:
- Daughters with `SesEndDate = NULL` were marked but NOT yet notified
- Daughters with `SesEndDate` set were already processed (redirect sent)
- All daughters with `SesEndDate = NULL` → SLO never started processing daughters
- Some daughters with `SesEndDate` set → SLO chain broke at a specific daughter

## Trace Activation Verification (combined)
```sql
SELECT 'SysPar' AS Source, SysParId AS Key_, SysParVal AS Value_
FROM gam.SysPar WHERE SysParId = 'EnableTracing'
UNION ALL
SELECT 'RepositoryProp', RepPropId, RepPropVal
FROM gam.RepositoryProp WHERE RepPropId = 'EnableTracing';
```

Both rows should return `'1'` for tracing to be active. Remember that the third switch (log.config root level) is file-based, not SQL-queryable
