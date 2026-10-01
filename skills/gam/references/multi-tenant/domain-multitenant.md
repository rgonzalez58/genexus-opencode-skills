---
name: domain-multitenant
description: Multi-tenant architecture — repository isolation, namespace routing, connection management
---

# GAM Multi-Tenant & Repository Encyclopedia
## Overview
GAM supports multi-tenancy through Multiple Repositories within a single GAM database. Each repository has its own users, roles, permissions, security policies, and applications — while sharing the same physical database

## Key Concept: Repository
A Repository is the logical boundary for tenant isolation. Each repository has:
- Its own `RepositoryGUID` (e.g., `<repository-guid>`)
- Its own `NameSpace` (e.g., `Test_Issues18u14`)
- Own users, roles, permissions, and security policies
- One or more Applications (each with its own `ApplicationId`)
- One or more Repository Connections

## Multi-Tenant Flag
```json
{"isGAMMultiTenant": false, "Version": "4.1.5", ...}
```
The `isGAMMultiTenant` flag in `GAMGetGAMGeneralSetting` determines if the GAM installation operates in multi-tenant mode

## Multi-Repository Scenarios
### Scenario 1: Multi-Tenant Application
The same application installation is shared by many companies. Each company = one Repository. Users are different per tenant. The app determines which repository to use at login time (e.g., via URL subdomain, login form field, etc.)

### Scenario 2: Company with Branches
One company with multiple branches. Users are defined ONCE in the GAM DB but have different roles/permissions per branch-repository. No user duplication needed

## Repository Connections
### `connection.gam` File
- The `connection.gam` file only contains the Key (a symmetric encryption key)
- Connections themselves are stored in the database, NOT in the file
- The Key in `connection.gam` can have N connections associated in the database
- Format: XML containing the key used to encrypt/decrypt connection credentials

### Connection Properties (from traces)
```json
{
	"Key": "<connection-key-guid>",
	"Name": "MyKBName",
	"Repository": "<repository-guid>",
	"UserName": "<conn-user>",
	"UserPassword": "<encrypted-password-base64>",
	"Type": "LAN",
	"Trusted": false,
	"Expires": true
}
```

Connection properties reference:
- `Key`
	* Unique connection identifier (GUID)
- `Name`
	* Human-readable connection name
- `Repository`
	* GUID of the repository this connection belongs to
- `UserName`
	* Connection user name (used for authentication to GAM)
- `UserPassword`
	* Encrypted password
- `Type`
	* Connection type (`LAN`, `WAN`, etc.)
- `Trusted`
	* If true, no password required
- `Expires`
	* If true, connection can expire

### Connection Resolution in Traces
```
GAMTrace-GAMGetRepositoryConnection - &CacheConnectionCli:{"Key":"...", "Name":"...", ...}
GAMTrace-GAMGetRepositoryConnection - &Errors:[]
```

### Environment Variable for Connections
Since GX 18u4, you can use an environment variable instead of `connection.gam`:
- Set `GAM_CONNECTION_KEY` environment variable with the Key value
- Avoids needing the physical `connection.gam` file in deployment

## `application.gam` File
- Stores the `ApplicationId` for the current application
- Required at runtime to identify which application within the repository is being accessed
- Setting: `GAMGetGAMGeneralSetting` returns `ApplicationId` from this file

## Cache Behavior
GAM caches repository info aggressively. The cache key is the `RepositoryGUID`

Trace signatures for cache operations:
```
GAMTrace-GAMGetCache - Cache name:com.genexus.gam.repositories   Key:<GUID>
GAMTrace-GAMSetCache - RepId:<namespace>:   Key:<GUID>   Value:{...}
GAMTrace-GAMGetCacheRepository - 1 &CacheRepository:<GUID>
```

Cache contains repository configuration including Id, GUID, Key, NameSpace, Name, Description, and connection details

## Shared-DB Scenario: IDP + Client in Same Database
A common deployment has the SSO IDP and client apps pointing to the same GAM database. This is valid but has a critical pitfall:

### `LoginTmp` State Collision
Both the client-side and IDP-side state handlers write to the same `LoginTmp` table with the same OAuth `state` key, causing a PRIMARY KEY violation

The fix: The IDP-side handler must prefix the key with `IDP-` so entries do not collide:
- Client writes: `GRESTD<hash>`
- IDP writes: `IDP-GRESTD<hash>`

Affected versions: u13HF and u14 removed the `IDP-` prefix. Fixed in u14HF

Error signature in logs:
```
ERROR … PRIMARY KEY violation … gam.LoginTmp
```

Workaround: Separate GAM databases for IDP and client apps

See `domain-session-token.md` LoginTmp section for full details

## Common Multi-Tenant Issues
- Wrong Repository on login: `connection.gam` key does not match any connection in DB, `GAMGetRepositoryConnection` returns errors
- Namespace mismatch: Application namespace is not equal to connection namespace, session creation fails
- Cache stale: After changing repository config, cached values persist. Restart app server or clear cache
- ApplicationId mismatch: `application.gam` has wrong `ApplicationId`, user gets wrong roles/permissions
- Shared-DB state collision: IDP and Client in same DB, `LoginTmp` duplicate key if `IDP-` prefix is missing (u13HF/u14 bug)

---

## Decision Points
Before configuring multi-tenant or diagnosing tenant issues, Claude MUST check which Decision Points apply

### DP-1: Architecture — Single vs Multi Repository
- Trigger: User asks about multi-tenant setup or tenant configuration
- Question: "Is your scenario: `single-repo` (one app, one repository, multiple tenants by namespace) or `multi-repo` (multiple repositories with separate or shared GAM databases)?"
- Options:
	* `single-repo` — One GAM repository, namespaces to differentiate tenants. Simpler, less isolation
	* `multi-repo` — (DEFAULT) Multiple repositories. Each tenant can have its own DB or share a DB with namespace routing
- Impact: Changes how `connection.gam` is configured, namespace routing, and the risk of Shared-DB collision
- Phase: design

### DP-2: Shared DB or Separate DBs?
- Trigger: When DP-1 = multi-repo
- Question: "Do the repositories share the same GAM database (`shared-db`) or does each one have its own DB (`separate-db`)?"
- Options:
	* `separate-db` — (DEFAULT) Each repo in its own DB. Maximum isolation. No risk of LoginTmp collision
	* `shared-db` — Repos in the same DB with different namespaces. Risk of `LoginTmp` state collision if IDP and client share the DB (see Shared-DB Scenario in this skill)
- Impact: `shared-db` requires verifying the IDP uses the `IDP-` prefix in `LoginTmp` keys. Affects required GAM version (u14HF+ for fix)
- Phase: design

### DP-3: Connection Key Distribution
- Trigger: When configuring deployment for multi-tenant
- Question: "How do you distribute the connection key to each tenant? Via `connection.gam` file, via `env var` (GAM_CONNECTION_KEY), or via `code`?"
- Options:
	* `file` — (DEFAULT) `connection.gam` file in the web root. Simple but requires deploy per tenant
	* `env_var` — `GAM_CONNECTION_KEY` environment variable. Useful for containers/Docker
	* `code` — Set connection key programmatically at startup. Maximum flexibility
- Impact: Affects the deployment method and how tenant routing is managed at runtime
- Phase: deploy
