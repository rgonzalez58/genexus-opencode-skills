---
name: kb-setup-troubleshooting
description: Errors seen during GAM KB setup and their fixes; login failures, build cancellations, `gx` import errors, MSBuild pitfalls, log locations
---

# GAM KB Setup: Troubleshooting Catalog
Scope: error messages seen during KB creation, text-format import, build, database creation, IIS deployment, and first-run smoke tests. Each entry lists the cause and the fix

Related files:
- [GAM KB Setup: Orchestrator (GAM scope only)](domain-kb-setup.md): orchestrator hub and layer model
- [GAM KB Setup: Fresh GAM Recipe (GAM-specific steps)](setup-recipe.md): fresh GAM step-by-step (errors seen here most often)
- [Connecting a New KB to an Existing GAM Database (GAM-specific recipe)](connect-existing-gam-db.md): existing GAM DB recipe (errors specific to this flow)
- [GAM Debugging Core](../debugging/common-debugging.md): runtime trace diagnosis (not build)

---

## Build and Database Connectivity

### `Login failed for user '<UserName>'`
- Cause: DataStore missing `TRUSTED_CONNECTION=Yes` (Windows Auth) or `USER_ID` / `USER_PASSWORD` (SQL Auth)
- Fix: set `UseTrustedConnection = 'Yes'` in `<EnvName>.local.env.gx` for Windows Auth, or set `UserId` and `UserPassword` for DBMS Auth, then re-import via `import_text_to_kb`

### `Memory stream is not expandable`
- Cause: corrupted internal DataStore configuration storage (size metadata mismatch)
- Fix: re-import `<EnvName>.local.env.gx` via `import_text_to_kb` to regenerate the internal storage cleanly. If the issue persists, delete the KB environment preferences and re-import both `<EnvName>.env.gx` and `<EnvName>.local.env.gx`

### `The file '…already exists'` during build
- Cause: residual files from a previous partial build
- Fix: close KB → delete `NetModel/web/` entirely → reopen → rebuild

### `Build canceled / Operation Canceled by the user`
- Cause: GAM initialization failed (check the log for the REAL error above this message)
- Fix: look at `GXMBLServices.log` for the actual error (usually login failure or blob deserialization)

### `Repository not found (GAM2)` at "Creating connection.gam file…"
- Cause: KB Repository GUID does not match any `RepGUID` in the existing GAM DB. Happens when a NEW KB is pointed at an EXISTING GAM DB without setting `RepositoryId`; GeneXus generates a random GUID the existing DB does not know
- Fix: set `RepositoryId` in `<EnvName>.local.env.gx` `#Properties` with the existing `RepGUID` (`SELECT RepGUID FROM gam.Repository WHERE RepId = 2`), then `import_text_to_kb({names: ["<EnvName>"], forceSave: true})` and rebuild. See [Recipe Steps](connect-existing-gam-db.md#recipe-steps)
- Verify after fix: `InputParms.json` contains `"Repository":"<existing-GUID>"` and `OutputParms.isOK:true`

### `TXP0007: Version X not found` during import
- Cause: wrong version name in the `.gx` file
- Fix: read the actual version name from the `Name` property inside the `#Version` block of `<KBName>.kb.gx`; usually `Design`, not the KB name

### `NullReferenceException` during import of version properties
- Cause: using a legacy `GAM`-prefixed property name that the current text format rejects
- Fix: use `AdministratorUserName`, not `GAMAdministratorUser`. See [Property Names That CAUSE ERRORS](domain-kb-setup.md#property-names-that-cause-errors) for the full list

### GAM property accepted at import but ignored at build
- Cause: the property was written as a quoted display name with spaces, such as `"Administrator User Name"` or `"Login Object for Web"`. The import succeeds, the property is dropped, and nothing warns
- Symptom: GAM init fails with a message unrelated to the property, or GAM behaves as if the property was never set
- Fix: rewrite every GAM property name in PascalCase, unquoted. See [GAM Activation Snippet](setup-recipe.md#gam-activation-snippet) for the exact text
- Detection: search the `.gx` files for a quoted property name containing a space; there should be none

## IIS Deployment

### `Login failed for user 'IIS AppPool\DefaultAppPool'` (HTTP 500)
- Cause: IIS AppPool identity lacks SQL Server login or database-level access. This is the #1 most common issue when deploying via IIS with Trusted Connection
- Fix: grant `IIS AppPool\DefaultAppPool` login + `db_owner` on BOTH Default and GAM databases. See [AppPool Permissions on the GAM Database](setup-recipe.md#apppool-permissions-on-the-gam-database)
- Detection: not visible in browser. Check `<TargetPath>/web/logs/stdout_*.log` (.NET Core) or Windows Event Log (.NET Framework)

### `Cannot open database "X"` with correct credentials
- Cause: database name mismatch; `appsettings.json` / `web.config` says one name, actual SQL DB has another (often `GX_KB_` prefix discrepancy)
- Fix: compare `Connection-<store>-DB` in config vs `SELECT name FROM sys.databases`. Fix in KB Environment DataStore properties for a permanent fix

## `gx` Import Errors

### `gx create-knowledge-base` returns "Must specify template"
- Cause: invalid `environment` value. Only `".NET"` or `"Java"` are valid
- Fix: do NOT use `"C#"`, `".NET Framework"`, or `"CSharp"`

### `gx create-knowledge-base` returns "Database already exists"
- Cause: attempting to recreate a KB over an existing directory / database
- Fix: drop the SQL database first (`DROP DATABASE [GX_KB_<name>]`), then delete the directory, then recreate

### `gx` session expired during build
- Cause: `gx` has a ~2 minute session timeout. Long builds exceed this
- Fix: build continues in background. Check `GXMBLServices.log`. For reliable builds, use MSBuild instead of `gx build-all`

### `gx import-text-to-kb` returns "No objects found" with type filter
- Cause: type filters (`type:version`, `type:environment`) do NOT work for preferences
- Fix: use object names instead; `names: ["Design", ".NET"]`. Always use `forceSave: true` to overwrite

## MSBuild

### MSBuild `Version 'Design' does not exist`
- Cause: using `Design` as `-p:ActiveVersion`. MSBuild resolves versions by KB name, not by model name
- Fix: use the KB name (e.g., `MyKB`) as `-p:ActiveVersion`

## Log Locations

### .NET Core Logs
- stdout logs: `<TargetPath>/web/logs/stdout_*.log` (directory may need manual creation)
- GeneXus `client.log`: `<TargetPath>/web/bin/client.log` (NOT in `web/` root)
- IIS logs: `%SystemDrive%\inetpub\logs\LogFiles\W3SVC1\` (requires admin to read)

### GXMBLServices Log
`<GXInstallPath>\GXMBLServices.log`: UTF-16 encoded, wide characters with null bytes between chars

Read with:

```bash
cat -v file.log | sed 's/\^@//g'
```
