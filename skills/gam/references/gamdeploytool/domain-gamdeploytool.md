---
name: domain-gamdeploytool
description: GAM Deploy Tool (agamdeploytool) command line — actions, global and per-action flags, per-platform invocation (Java / .NET Framework / .NET Core), connection.gam handling, XML config mode, cross-generator peculiarities, and troubleshooting
---

# GAM Deploy Tool (`agamdeploytool`)
Command-line utility that performs GAM deployment operations: initialize/upgrade the GAM database metadata, import/export repository packages (`.gpkg`), create/delete repositories, and manage `connection.gam`. Ships as `GAMDeployTool.zip` inside `Library/GAM/Platforms/<platform>/` and is unzipped where it runs

Available since GeneXus 16 Upgrade 3. The [official docs](https://docs.genexus.com/en/wiki?37764,GAM+Deploy+Tool+command+line+%28Windows+and+Unix-like+operating+systems%29) are versioned per GeneXus release; this reference targets the latest (GeneXus 18 Upgrade 15). Actions, flags, and behavior may differ in earlier versions — default to the latest, and when in doubt direct the user to the wiki page for the specific GeneXus version they run

Reuses `connection.gam` concepts documented in [GAM Connection & Configuration Encyclopedia](../multi-tenant/domain-connection-config.md) — read it for the file structure and `GAM_CONNECTION_KEY`

---

## What it does — and does NOT do
Does:
- Initialize the GAM database metadata (`-initialize`) and update the metadata version (`-upgradegam`)
- Export a repository to a `.gpkg` package and import it into another environment
- Create and delete repositories
- Generate, read, and update `connection.gam`

Does NOT:
- Create or reorganize GAM database tables — that is the DBA's responsibility using standard DB reorganization
- `-upgradegam` runs AFTER the DBA reorganizes the tables; it only aligns the GAM metadata version, it does not alter the schema

---

## Invocation per platform
Run from the folder where `GAMDeployTool.zip` was unzipped. Flags may be passed as one quoted string or as separate tokens

- Java: `java -cp ./* genexus.security.api.agamdeploytool "<action> <flags>"` — cross-platform; `-cp ./*` loads the tool jars and JDBC drivers
- .NET Framework: `agamdeploytool.exe "<action> <flags>"` — Windows only; run from `bin`
- .NET Core: `dotnet agamdeploytool.dll "<action> <flags>"` — cross-platform; run from `bin`

---

## Prerequisites (copy next to the tool)
- `client.cfg` — Java — database connection settings
- `client.exe.config` — .NET Framework / .NET Core — database connection settings
- `application.key` — all — encryption key
- `log.config` / `log4j2.xml` — optional — enable tracing/diagnostics
- DBMS driver — as needed — JDBC jar (Java) or provider (.NET) matching the engine

The database must be reachable and the GAM tables must already exist (created and reorganized by the DBA) before running any action

---

## Actions
Every action additionally requires `-admin_name` / `-admin_pass` on top of its own required flags listed below

- `-initialize` — initialize GAM DB metadata (first-time) — requires `-connection_gam_file_path`
- `-upgradegam` — update GAM metadata version after DBA reorg — requires `-connection_gam_file_path`
- `-import` — import a `.gpkg` package into a repository — requires `-file_path_package`, `-connection_gam_file_path`
- `-export` — export a repository to a `.gpkg` package — requires `-target`, `-rep_guid`, `-pkg_name`
- `-new_rep_create` — create a new repository — requires `-connection_gam_file_path` + `-new_rep_*` (below)
- `-delete_rep` — delete a repository — requires `-rep_guid`, `-rep_name`
- `-getconnections` — list connections grouped by repository from `connection.gam` — no extra required flags
- `-updateconnectionfile` — create/update `connection.gam` entries — requires `-target`, `-connections`
- `-generatexml` — print an example XML config for `-xml_config_file` — no extra required flags
- `-xml_config_file <path>` — load ALL parameters from an XML file (exclusive) — requires `<path>`
- `-help` — print the flags supported by the chosen action — no extra required flags

---

## Global flags
- `-admin_name` `<gam_admin_user>` — GAM administrator user (example: `gamadmin`)
- `-admin_pass` `<password>` — GAM administrator password
- `-connection_gam_file_path` `<dir-or-file>` — path to `connection.gam` (default `./connection.gam`)
- `-connection_key` `<key>` — encryption key written to / read from `connection.gam`
- `-verbose` `true` \| `false` \| `debug` — console verbosity; `debug` adds diagnostic detail
- `-xml_config_file` `<path>` — read all parameters from XML (bypasses CLI flags)
- `-help` — print flags for the chosen action (no value)

---

## Per-action flags
### `-new_rep_create` (also usable as `-new_rep_create true` inside `-import`)
- `-new_rep_name` — required — repository name
- `-new_rep_namespace` — required — repository namespace
- `-new_rep_admin_name` — required — repository administrator user
- `-new_rep_admin_pass` — required — repository administrator password
- `-new_rep_conn_usr_name` — required — connection user for the repository
- `-new_rep_conn_usr_pass` — required — connection user password
- `-new_rep_guid` — optional — custom repository GUID (auto-generated if omitted)
- `-admin_role_guid` — optional — administrator role GUID (import context)
- `-connection_key` — optional — encryption key written to `connection.gam`

### `-delete_rep`
- `-rep_guid` — required — GUID of the repository to delete
- `-rep_name` — required — repository name (confirmation guard)

### `-import`
- `-file_path_package` `<path.gpkg>` — package to import
- `-imp_full` `true` \| `false` — import everything
- `-upd_rep` / `-upd_rep_guid` `true` / `<guid>` — update an existing repository / its GUID
- `-new_rep_create` `true` \| `false` — create the target repository during import (with `-new_rep_*`)
- `-imp_sec_policies` `true` \| `false` — import security policies
- `-imp_users` `true` \| `false` — import users
- `-imp_roles` `true` \| `false` — import roles
- `-imp_auth_types` `true` \| `false` — import authentication types
- `-imp_eve_subscriptions` `true` \| `false` — import event subscriptions
- `-imp_connections` `true` \| `false` — import connections
- `-imp_apps` `full` \| `custom` \| `none` — applications import mode
- `-imp_apps_details` `<AppGuid>,<ImportPrms>;…` — app list when `-imp_apps custom`
- `-disable_upd_role_prm` `true` \| `false` — do not update role permissions on import

### `-export`
- `-target` `<dir>` — output directory for the package
- `-rep_guid` `<guid>` — repository to export
- `-pkg_name` `<name>` — output package name
- `-full_export` `true` \| `false` — export all data (sets the per-entity flags)
- `-exp_users` `true` \| `false` — export users
- `-exp_roles` `true` \| `false` — export roles
- `-exp_eve_subscriptions` `true` \| `false` — export event subscriptions
- `-apps` `<guid,guid,…>` — application GUIDs to export
- `-roles` `<selection>` — custom roles selection

### `-updateconnectionfile`
- `-target` `<dir>` — directory where `connection.gam` is written
- `-connections` `<Repository>,<ConnectionName>;…` — repository/connection pairs (case-sensitive)

`-getconnections` and `-generatexml` take no additional flags

---

## XML config mode
For long invocations (see the 50-parameter limit below), drive the tool from a file:

- Run `-generatexml` to print a template for the Export action
- Edit the template with the target parameters
- Run `-xml_config_file <path>` to execute

`-xml_config_file` is exclusive: when present, all CLI flags are ignored and every parameter comes from the XML

---

## connection.gam handling
- `-connection_gam_file_path` selects the file (default `./connection.gam`); `-connection_key` supplies its encryption key
- `-import` and `-new_rep_create` produce or update `connection.gam` for the created repository
- `-getconnections` lists the stored connections; `-updateconnectionfile` writes/updates entries
- The runtime needs this file (or `GAM_CONNECTION_KEY`) to reach the GAM DB — see [GAM Connection & Configuration Encyclopedia](../multi-tenant/domain-connection-config.md)

---

## Cross-generator peculiarities
- `.NET Framework` is Windows only; `.NET Core` and `Java` are cross-platform
- Config file name differs by generator: `client.cfg` (Java) vs `client.exe.config` (.NET). A mismatched file means the tool cannot read the DB connection
- Path handling during `-import` detects the OS to choose the separator (`\` on Windows, `/` on Unix). This detection relies on a Java-native call; on .NET generators no equivalent branch is observed, so it defaults to Windows-style separators — verify `-file_path_package` / `-target` paths when running the .NET Core build on Unix (observed in the beta build; treat as version-specific)
- Java loads JDBC drivers from the classpath (`-cp ./*`); .NET uses the provider configured in `client.exe.config`

---

## Gotchas
- Max 50 positional parameters — use `-xml_config_file` for longer runs
- A flag value containing spaces ends at the next `-flag`; avoid spaces in values or use the XML config
- `-connections` preserves case (matters for case-sensitive engines such as PostgreSQL); other flags match case-insensitively
- No documented numeric exit codes — read the stdout status messages; enable `-verbose debug` (plus `log.config` / `log4j2.xml`) to capture the failing step
- `-upgradegam` does not reorganize tables; run it only after the DBA completes the DB reorganization

---

## Recipes
Initialize a fresh GAM DB (Java):
```
java -cp ./* genexus.security.api.agamdeploytool "-initialize -admin_name <admin> -admin_pass <pass> -connection_gam_file_path ./connection.gam -verbose true"
```

Upgrade GAM after the DBA reorganized tables (.NET Core):
```
dotnet agamdeploytool.dll "-upgradegam -admin_name <admin> -admin_pass <pass> -connection_gam_file_path ./connection.gam"
```

Export a repository to a package (.NET Framework):
```
agamdeploytool.exe "-export -admin_name <admin> -admin_pass <pass> -target C:\export -rep_guid <repo_guid> -pkg_name mypackage -full_export true"
```

Import a package into a new repository (Java):
```
java -cp ./* genexus.security.api.agamdeploytool "-import -admin_name <admin> -admin_pass <pass> -file_path_package /pkgs/mypackage.gpkg -new_rep_create true -new_rep_name <RepoName> -new_rep_namespace <Namespace> -new_rep_admin_name <repo_admin> -new_rep_admin_pass <repo_pass> -new_rep_conn_usr_name <conn_user> -new_rep_conn_usr_pass <conn_pass> -imp_connections true -connection_gam_file_path ./ -connection_key <key>"
```

Delete a repository (.NET Core):
```
dotnet agamdeploytool.dll "-delete_rep -admin_name <admin> -admin_pass <pass> -rep_guid <repo_guid> -rep_name <RepoName>"
```

Read / update `connection.gam` (Java):
```
java -cp ./* genexus.security.api.agamdeploytool "-getconnections -admin_name <admin> -admin_pass <pass>"
java -cp ./* genexus.security.api.agamdeploytool "-updateconnectionfile -admin_name <admin> -admin_pass <pass> -target ./ -connections <Repository>,<ConnectionName>"
```

---

## Troubleshooting checklist
- Config files present next to the tool (`client.cfg` / `client.exe.config` + `application.key`)?
- DB reachable and GAM tables already created + reorganized by the DBA? (`-initialize` / `-upgradegam` never create or reorg tables)
- DBMS driver available (JDBC jar in classpath for Java; provider for .NET)?
- Correct action and its required flags supplied?
- `connection.gam` path and `-connection_key` correct?
- Enable `-verbose debug` and the tracing files to capture the failing step

---

## See also
- [GAM Deploy Tool command line (Windows and Unix-like)](https://docs.genexus.com/en/wiki?37764,GAM+Deploy+Tool+command+line+%28Windows+and+Unix-like+operating+systems%29)
- [GAM Connection & Configuration Encyclopedia](../multi-tenant/domain-connection-config.md)
- [GAM KB Setup — Orchestrator (GAM scope only)](../kb-setup/domain-kb-setup.md)
- [GeneXus Environment: Java](../environments/environment-java.md) · [GeneXus Environment: .NET Core / .NET 5+](../environments/environment-netcore.md) · [GeneXus Environment: .NET Framework](../environments/environment-netframework.md)
