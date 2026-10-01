---
name: domain-connection-config
description: connection.gam structure, application.gam, environment variables, deployment patterns
---

# GAM Connection & Configuration Encyclopedia

## Overview
GAM uses two primary configuration files at runtime:
- `connection.gam`: Contains the encryption key for repository connections
- `application.gam`: Identifies the application within the repository

## `connection.gam`

### Purpose
The `connection.gam` file contains a symmetric encryption key used to look up repository connections stored in the GAM database. It does NOT contain the connections themselves

### Structure
```xml
<GAMConnection>
	<Key><connection-key-guid></Key>
</GAMConnection>
```

### How it works
- Application starts → reads `connection.gam` → gets the Key
- Key is used to query the `SysConnectionConfig` table for matching rows (one Key can map to multiple repositories)
- Each matching row provides UserName, encrypted UserPassword, repository binding, and connection properties
- The runtime then validates those credentials against the `GAMRepositoryConnection` table for the same repository and connection name. See the Three-layer architecture section below

### Generate via Backoffice
In GAM Web Backoffice → Repository → Connections:
- USE CURRENT KEY: Associates a new connection with the existing key
- USE AUTOMATIC KEY: Generates a new key and connection.gam content
- FILE button: Exports the XML for the `connection.gam` file

> The Backoffice exposes two screens with distinct scopes; do not confuse them:
> - **Repository → Connections (edit row)**: manages credentials; UserName, UserPassword, and Encryption Key
> - **Repository → Connection Keys**: manages the GUID written to `connection.gam`

### Environment Variable Alternative (GeneXus 18u4+)
Set `GAM_CONNECTION_KEY` environment variable with the Key GUID value:
```
GAM_CONNECTION_KEY=<connection-key-guid>
```
This eliminates the need for the physical `connection.gam` file. When both are present, the env var takes priority

### Three-layer architecture: file, SysConnectionConfig, GAMRepositoryConnection
GAM splits connection state across three artifacts that must stay aligned:

- `connection.gam` file (or `GX_GAMConnectionKey` env var, which has priority). Single field, a Key (GUID). Lives in the deployment
- `SysConnectionConfig` table, in the GAM database. Keyed by `(SysConnCfgKey, SysConnCfgRep, SysConnCfgName)`. Each row holds `SysConnCfgUser`, `SysConnCfgPwd` (encrypted), and `SysConnCfgJson` (Type, Language, dynamic properties). Queried at runtime by `SysConnCfgKey = <key from connection.gam>` to materialize the deployment's view of which connections it is allowed to use
- `GAMRepositoryConnection` table, in the GAM database. Keyed by `(RepId, RepConName)`. Each row holds `RepConUser`, `RepConPwd` (encrypted), `RepConKey` (the encryption key used for that connection's password), and `RepConLng`. This is the repository's authoritative registry of accepted connections

Relationship between the three. The Key in `connection.gam` selects one or more rows in `SysConnectionConfig` (the same Key can map to multiple repositories). For each row resolved there, the pair `(SysConnCfgRep, SysConnCfgName)` locates the corresponding row in `GAMRepositoryConnection`. The runtime then compares `SysConnCfgUser`/`SysConnCfgPwd` against `RepConUser`/`RepConPwd`. Both must match for the request to proceed

Failure modes when the layers diverge. If the Key has no rows in `SysConnectionConfig`, GAM raises `ConnectionNotFound`. If the user differs between tables, `ConnectionLoginFailed`. If the decrypted password differs, `ConnectionPasswordFailed`. Any of those aborts `GAMGetRepositoryConnection`, which is called by every login, session check, and permission evaluation, so the deployment becomes unusable until the layers are aligned again

How the layers stay aligned. Both tables are populated together when a repository is created (`GAMRepositoryNewInternal` writes to `GAMRepositoryConnection` and `SaveSysConnectionConfigs` writes to `SysConnectionConfig`) and kept aligned by the public flows. The Backoffice blocks editing or deleting the connection currently in use (errors `CanNotChangeCurrentConnection` / `CanNotDeleteCurrentConnection`) precisely to prevent these two views from diverging

Field naming pitfall. `GAMRepositoryConnection` does not have `Trusted` or `Expires` columns. Those names appear in some traces because the runtime SDT `CacheConnectionCli` includes them, but `Trusted` is hardcoded to `False` and there is no Trusted Connection switch at the GAM layer

### Encryption Key (per-connection credential key)
The **Encryption Key** field on the connection credentials screen (Repository → Connections) is a per-connection key GAM uses to protect credentials stored in `SysConnectionConfig`:

- Each connection has its own independent Encryption Key
- GAM uses it to encrypt the connection's UserName and UserPassword before writing them to `SysConnectionConfig`
- The GUID in `connection.gam` is a separate value: it is the lookup key for `SysConnectionConfig`, generated via the Connection Keys screen
- Use the **Generate** button on the form to create a random secure key: recommended for new connections
- Changing the Encryption Key of an existing connection invalidates its `SysConnectionConfig` entry; any deployment whose `connection.gam` references that connection stops resolving until regenerated via the Connection Keys screen
- Changing the Encryption Key of the **currently active connection** raises Error 127 `CanNotChangeCurrentConnection`: put the application in maintenance mode first

## `application.gam`

### Purpose
Identifies which application (within a repository) the current deployment represents

### Key Fields
- `ApplicationId`: Numeric ID of the application
- Used by `GAMGetGAMGeneralSetting` to determine application context

## Connection Management via Code (`GAM` External Object)

### Setting the Active Connection
In multi-tenant or multi-repository scenarios, use `GAM.SetConnection()` to select the active repository connection before calling other GAM objects:

```genexus
// Set connection by name
&Ok = GAM.SetConnection(!"TenantA", &Errors)
If not &Ok
	// Connection name not found or invalid
EndIf
```

### Listing Available Connections
```genexus
// Get all available connections for the current key
&Connections = GAM.GetConnections()
For &ConnectionInfo in &Connections
	// &ConnectionInfo.Name: connection name
	// &ConnectionInfo.RepositoryName: repository name
	// &ConnectionInfo.UserName: connection username
EndFor
```

### Example: Resolve Tenant Connection
```genexus
&ConnectionInfos = GAM.GetConnections()
For &ConnectionInfo in &ConnectionInfos
	If &ConnectionInfo.Name = &CompanyId
		&Ok = GAM.SetConnection(&ConnectionInfo.Name, &Errors)
		Exit
	EndIf
EndFor
```

### `GAMConnectionInfo` Properties
- `Name`
	* Connection name (value passed to `SetConnection`)
- `RepositoryName`
	* Repository name associated to this connection
- `UserName`
	* Connection username

### Runtime Resolution Order
GAM resolves the connection key in this order:
- Cache: if already resolved in this session
- `GX_GAMConnectionKey` environment variable: preferred for containerized deployments
- `connection.gam` file: standard file-based deployment

### Constraints: SetConnection
- Call `SetConnection` before any other GAM object call when the connection set has more than one entry
- Omitting `SetConnection` in a multi-tenant environment causes a runtime error
- In single-connection scenarios, `SetConnection` is not needed: GAM auto-resolves

---

## Deployment Patterns

### Single App, Single GAM DB
```
App → connection.gam (Key) → GAM DB → Repository → Application
```

### Multiple Apps, Shared GAM DB
```
App A → connection_a.gam (Key A) → GAM DB → Repo → App A (Id=2)
App B → connection_b.gam (Key B) → GAM DB → Repo → App B (Id=4)
```
Best practice: One connection per application for security and performance

### Multi-Tenant (Multiple Repositories)
```
Tenant 1 → connection.gam → GAM DB → Repository 1
Tenant 2 → connection.gam → GAM DB → Repository 2
```
Login determines which repository via namespace/subdomain routing

## Trace Patterns
```
GAMTrace-GAMGetRepositoryConnection - …{"Key":"<GUID>","Name":"<NAME>","Repository":"<REPO_GUID>","UserName":"<USER>","UserPassword":"<ENCRYPTED>","Type":"LAN","Trusted":false,"Expires":true}
GAMTrace-GAMGetRepositoryConnection - &Errors:[]
```

The dump above is the SDT `CacheConnectionCli` after resolution. `Key`, `Repository`, `Name`, `UserName`, `UserPassword` and `Type` come from `SysConnectionConfig` (the row matched by the Key). `Trusted` and `Expires` are SDT-only fields not stored in any GAM table; `Trusted` is hardcoded to `False`

Errors appear when:
- `connection.gam` Key has no rows in `SysConnectionConfig` → `ConnectionNotFound`
- `SysConnCfgUser` does not match `RepConUser` for the same repository and connection name → `ConnectionLoginFailed`
- The decrypted `SysConnCfgPwd` does not match the decrypted `RepConPwd` → `ConnectionPasswordFailed`
- DB is unreachable

## Log Configuration for Debugging
When connection issues occur, you need to capture GAM traces to diagnose them
See [GAM Debugging Core](../debugging/common-debugging.md) for complete log configuration:

- .NET Framework: `web.config` `<system.diagnostics>` section to redirect traces to file
- .NET Core: `appsettings.json` `Logging` section + stdout redirect
- Java: `log4j.properties` or `logback.xml` configuration
- Diagnostic SQL: Direct queries to `GAMRepositoryConnection` table to verify key matches (see below)

The key trace to look for when debugging connection issues:
```
GAMTrace-GAMGetRepositoryConnection - …{"Key":"<GUID>","Name":"…"}
GAMTrace-GAMGetRepositoryConnection - &Errors:[…]
```

If this trace is ABSENT, traces may not be activated. See [Activating GAM Traces (3-Step Process)](../debugging/common-debugging.md#activating-gam-traces-3-step-process)

## Connecting a New KB to an Existing GAM Database

### When This Applies
A new KB is being created but the GAM database already exists (shared GAM across multiple apps). The build generates a NEW `connection.gam` with a NEW key that does NOT exist in the existing DB. This MUST be replaced

### Post-Build Mandatory Steps
- Get the existing connection key (one of these methods):
	* Diagnostic SQL: `SELECT RepConKey FROM gam.RepositoryConnection WHERE RepConName = '<name>'`
	* Backoffice: Repository → Connections → FILE button
	* User provides it directly
- Replace connection.gam in the deployed web directory:
	```xml
	<GAMConnection>
		<Key>{existing-key-guid-from-step-1}</Key>
	</GAMConnection>
	```
	Path: `{TargetPath}/web/connection.gam`
- Verify application.gam matches the existing ApplicationID:
	```sql
	-- Diagnostic SQL: verify application ID mapping
	SELECT GAMApId, GAMApNme FROM GAMApplication
	```
	If the ID differs, update `application.gam` accordingly
- Test: Login with existing admin credentials (NOT a shared/default password)

### What Happens If You Skip This
- `connection.gam` has a key that doesn't exist in the GAM DB
- GAM trace: `GAMTrace-GAMGetRepositoryConnection - &Errors:[{…no connection found…}]`
- Login fails silently or shows "Repository not found" errors

### Cross-Reference
See [Recipe Steps](../kb-setup/connect-existing-gam-db.md#recipe-steps) in `connect-existing-gam-db.md` for the complete recipe

## Common Issues
- "No connection found": `connection.gam` key doesn't match any entry in the GAM DB. Regenerate via Backoffice. Most common when connecting a new KB to an existing GAM DB; the build-generated key must be replaced
- Wrong application context: `application.gam` has incorrect `ApplicationId`. Verify in Backoffice → Applications
- Deployment without `connection.gam`: Use the `GAM_CONNECTION_KEY` env var instead
- Connection expired: `Expires=true` and the connection's expiration time has passed

---

## Decision Points
Before configuring connections or diagnosing connection issues, Claude MUST check which Decision Points apply

### DP-1: Connection Key Distribution Method
- Trigger: User asks about connection.gam, deployment config, or tenant routing
- Question: "How do you want to distribute the connection key? Via `file` (connection.gam), via `environment variable` (GAM_CONNECTION_KEY), or via `code`?"
- Options:
	* `file`: (DEFAULT) `connection.gam` file in the web/ directory. Simple, standard. One file per deployment
	* `env_var`: `GAM_CONNECTION_KEY` environment variable. Ideal for containers, Docker, cloud deployments
	* `code`: Set programmatically at startup. Maximum flexibility for dynamic multi-tenant
- Impact: Affects the deployment method and how the repository is resolved at runtime
- Phase: deploy

### DP-2: Scenario; Single App or Multi-App?
- Trigger: When configuring connections
- Question: "Is this a single application or multiple applications sharing the same GAM database?"
- Options:
	* `single`: (DEFAULT) One app, one connection.gam, one application.gam. Standard configuration
	* `multi`: Multiple apps pointing to the same GAM DB. Each app has its own connection.gam with the same Key but may have a different ApplicationId in application.gam
- Impact: In multi-app, verify that ALL apps use the same connection key but have different ApplicationIds. Common mistake: copying application.gam between apps without changing the ApplicationId
- Phase: design
