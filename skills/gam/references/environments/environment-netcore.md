---
name: environment-netcore
description: .NET Core environment; Kestrel, appsettings.json, Docker deployment
---

# GeneXus Environment: .NET Core / .NET 5+

## Identity
- Generator (text format): `.NET`
- Language (text format): `.NET`
- Generator ID: 41
- Model ID: 45
- GXSPC directory: GXSPC045
- Build targets: DotNetCoreBaseProject.targets
- VER file pattern: `GNetCoreB<build>.VER`
- PRO file pattern: `GX_PRO41.045`

---

## Environment Text Format

### Main (`{EnvName}.env.gx`)
```
Environment {EnvironmentName}
{
	#Backend
		Generators
		{
			Default
				[
					Generator = '.NET'
				]
		}
		DataStores
		{
			Default
				[
					Dbms = 'SQL Server'
				]
		}
	#End

	#Properties
		UserInterface = "Web"
		Language = ".NET"
		DataSource = "SQL Server"
		TargetPath = "{TargetDir}"
		Name = "{EnvironmentName}"
	#End
}
```

### Local (`{EnvName}.local.env.gx`)
Same format as .NET Framework (see `environment-netframework.md`)

### GAM Environment-scope Properties
When this environment runs with GAM, add to the `#Properties` block of `{EnvName}.local.env.gx`; these carry secrets and never belong in the shared `{EnvName}.env.gx`:
- `AdministratorUserName`, `AdministratorUserPassword`: GAM init credentials
- `ConnectionUserName`, `ConnectionUserPassword`: Repository connection credentials
- `RepositoryId`: ONLY when connecting to an existing GAM DB

Write every name in PascalCase, unquoted; a quoted display name with spaces is ignored silently

See [GAM Activation Snippet](../kb-setup/setup-recipe.md#gam-activation-snippet) for the exact text, and [Recipe Steps](../kb-setup/connect-existing-gam-db.md#recipe-steps) for the existing-GAM recipe

---

## Runtime Config: `appsettings.json`
- Located: `{TargetPath}/web/appsettings.json`
- Format: JSON with `appSettings` and `languages` sections
- Connection values encrypted (base64): Datasource, User, Password, Schema
- DB name in plain text
- Also has `web.config` but only for IIS reverse proxy (not runtime config)

### Key Settings
```json
{
	"appSettings": {
		"EnableIntegratedSecurity": "1",
		"IntegratedSecurityLoginWeb": "gamexamplelogin",
		"IntegratedSecurityNotAuthorizedWeb": "gam_notauthorized",
		"SessionTimeout": "20"
	}
}
```

---

## GAM Init: `client.exe.config`
- Located: `{TargetPath}/Library/GAM/client.exe.config`
- Same XML format as .NET Framework
- `MODEL_NUM = 45`, `GENERATOR_NUM = 15` (uses .NET Framework tool for init)

### InputParms.json (Simplified)
```json
{
	"Applications": null,
	"FilesPath": null,
	"IdeConnection": null,
	"RepositoryGUID": "{repository_guid}"
}
```

Key difference: .NET Core uses simplified InputParms with only RepositoryGUID

---

## Build
Same MSBuild command as .NET Framework

### Unique Build Files
- `DotNetCoreBaseProject.targets` in build/
- `GeneXusSecurity.props` in build/
- `LastBuild.sln` in build/

### Unique Root Files
- `Directory.build.props` at environment root
- `NuGet.Config` at environment root
- `global.json` at environment root

---

## Deployment

### DEFAULT: IIS with Virtual Directory
This is the standard deployment method. Always use this unless explicitly told otherwise

.NET Core does NOT generate `VirtualDir.exe` (unlike .NET Framework). Must configure IIS manually

Virtual Directory naming convention: `{KBName}{EnvironmentSuffix}`
Where EnvironmentSuffix matches the DBMS combination: `NetCoreSQL`, `NetCoreMySQL`, `NetCoreD2C`, etc

Setup (requires elevated/admin terminal):
```powershell
Import-Module WebAdministration
New-WebApplication -Name "{KBName}{EnvSuffix}" -Site "Default Web Site" `
	-PhysicalPath "{KBPath}\{TargetPath}\web" -ApplicationPool "DefaultAppPool"
```

Or via appcmd:
```cmd
%systemroot%\system32\inetsrv\appcmd add app /site.name:"Default Web Site" /path:/{KBName}{EnvSuffix} /physicalPath:"{KBPath}\{TargetPath}\web"
```

Prerequisites: IIS installed with ASP.NET Core Hosting Bundle (ANCM module)

### CRITICAL: IIS AppPool SQL Server Permissions (Trusted Connection)
MUST DO after database creation. The IIS AppPool identity (`IIS AppPool\DefaultAppPool`) needs explicit SQL Server access to BOTH Default and GAM databases. Without this, the app returns HTTP 500 with no visible error in the browser

```sql
-- Run via sqlcmd -S localhost -E (or any sysadmin account)
IF NOT EXISTS (SELECT 1 FROM sys.server_principals WHERE name = 'IIS AppPool\DefaultAppPool')
	CREATE LOGIN [IIS AppPool\DefaultAppPool] FROM WINDOWS;

USE [{AppDB}];
IF NOT EXISTS (SELECT 1 FROM sys.database_principals WHERE name = 'IIS AppPool\DefaultAppPool')
	CREATE USER [IIS AppPool\DefaultAppPool] FOR LOGIN [IIS AppPool\DefaultAppPool];
ALTER ROLE db_owner ADD MEMBER [IIS AppPool\DefaultAppPool];

USE [{AppDB}_GAM];
IF NOT EXISTS (SELECT 1 FROM sys.database_principals WHERE name = 'IIS AppPool\DefaultAppPool')
	CREATE USER [IIS AppPool\DefaultAppPool] FOR LOGIN [IIS AppPool\DefaultAppPool];
ALTER ROLE db_owner ADD MEMBER [IIS AppPool\DefaultAppPool];
```

Error signature when missing: `DBMS Error Code:4060.Cannot open database "X" requested by the login. Login failed for user 'IIS AppPool\DefaultAppPool'.` (visible only in stdout logs, not in browser)

### Fallback: Kestrel Standalone (development only)
```bash
cd {KBPath}/{TargetPath}/web
dotnet bin/GxNetCoreStartup.dll
```

- Default port: 5000
- Entry point: `GxNetCoreStartup.dll` (NOT `GeneXus.Programs.Common.dll`)
- URL: `http://localhost:5000/gamexamplelogin.aspx`

### URL Patterns (IIS Virtual Directory)
```
http://localhost/{VirtualDir}/gamexamplelogin.aspx
http://localhost/{VirtualDir}/oauth/access_token
http://localhost/{VirtualDir}/.well-known/openid-configuration
```

### URL Patterns (Kestrel Standalone)
```
http://localhost:5000/gamexamplelogin.aspx
http://localhost:5000/oauth/access_token
http://localhost:5000/.well-known/openid-configuration
```

---

## Log Locations
- stdout
	* Path: `{TargetPath}/web/logs/stdout_*.log`
	* Notes: IIS stdout capture. Directory may need manual creation
- client.log
	* Path: `{TargetPath}/web/bin/client.log`
	* Notes: GeneXus runtime log. NOT in web/ root
- IIS logs
	* Path: `%SystemDrive%\inetpub\logs\LogFiles\W3SVC1\`
	* Notes: Requires admin to read
- GX build
	* Path: `{GXInstallPath}\GXMBLServices.log`
	* Notes: UTF-16 encoded

IMPORTANT: `web.config` references `stdoutLogFile=".\logs\stdout"`; if the `logs/` directory doesn't exist, stdout logging silently fails. Create it manually: `mkdir {TargetPath}/web/logs`

---

## Hosting Model
- Kestrel runs OutOfProcess behind IIS (default for GeneXus .NET Core deployments)
- IIS acts as a reverse proxy forwarding requests to Kestrel
- The Kestrel process is managed by the ASP.NET Core Module in IIS
- Recycle the IIS AppPool to restart Kestrel and force configuration reload (for example, after updating GAM trace switches: see [Cache Expiry Requirement](../debugging/common-debugging.md#cache-expiry-requirement))

---

## Session Cookies
- `.AspNetCore.Session.<appname>`: ASP.NET Core server-side session identifier
- `GAMSessionGUID`: GAM session identifier (persists across requests)

---

## stdout Logging Configuration
ASP.NET Core runs behind IIS via the ASP.NET Core Module. The `web.config` controls stdout logging:

```xml
<!-- Default: stdout logging DISABLED -->
<aspNetCore processPath="dotnet" arguments=".\bin\GxNetCoreStartup.dll"
						stdoutLogEnabled="false"
						stdoutLogFile=".\logs\stdout"
						hostingModel="OutOfProcess">
```

Enable for middleware-level diagnostics:

```xml
<aspNetCore processPath="dotnet" arguments=".\bin\GxNetCoreStartup.dll"
						stdoutLogEnabled="true"
						stdoutLogFile=".\logs\stdout"
						hostingModel="OutOfProcess">
```

Constraint: the `logs/` directory must exist before enabling; ASP.NET Core will NOT create it automatically. Create `<KB>/NetModel/web/logs/` manually

---

## Key Differences from .NET Framework
- Log config: .NET Framework uses `log.config` in `bin/` and `web/`; .NET Core uses the same `log.config` locations
- stdout: .NET Framework N/A (IIS direct); .NET Core uses `web.config` `stdoutLogEnabled`
- Session cookie: .NET Framework `ASP.NET_SessionId`; .NET Core `.AspNetCore.Session.<app>`
- Hosting: .NET Framework InProcess (IIS `w3wp.exe`); .NET Core OutOfProcess (Kestrel behind IIS)
- App restart: .NET Framework recycle AppPool; .NET Core recycle AppPool (kills Kestrel too)

---

## Troubleshooting

### HTTP 500: Login failed for 'IIS AppPool\DefaultAppPool'
Number 1 most common IIS deployment issue. See "IIS AppPool SQL Server Permissions" section above
Visible only in `{TargetPath}/web/logs/stdout_*.log`, not in browser

### DB name mismatch (GX_KB_ prefix)
GeneXus may create the database as `GX_KB_{KBName}` but `appsettings.json` has `Connection-Default-DB: {KBName}` (without prefix). Verify with:
```bash
sqlcmd -S localhost -E -Q "SELECT name FROM sys.databases WHERE name LIKE '%{KBName}%'"
grep "Connection-Default-DB" {TargetPath}/web/appsettings.json
```
Fix in KB Environment DataStore properties (permanent) or patch `appsettings.json` (temporary)

### Port 5000 in use
Use `--urls http://localhost:<port>` argument to change port

### Missing global.json
Causes SDK version mismatch. File is generated during build

### Cannot edit appsettings.json connections
Values are encrypted. Fix the `{EnvName}.local.env.gx` file and reimport via `import_text_to_kb` instead

---

## Decision Points
These decisions apply when this environment skill is active (user chose .NET Core). Present applicable ones before proceeding

### DP-1: Deploy Method
- Trigger: Before deploying the application (post-build)
- Question: "Deploy via `IIS Virtual Directory` (default, production) or `Kestrel standalone` (development only)?"
- Options:
	* `iis`: (DEFAULT) Virtual directory in IIS. Requires IIS + ASP.NET Core Hosting Bundle. Requires AppPool permissions if Trusted Connection
	* `kestrel`: Standalone on localhost:5000. Development/quick testing only. Does not require IIS
- Impact:
	* `iis`: URL = `http://localhost/{VirtualDir}/`. Requires AppPool SQL permissions step if Trusted Connection
	* `kestrel`: URL = `http://localhost:5000/`. No IIS config but not suitable for production
- Phase: deploy

### DP-2: Virtual Directory Name
- Trigger: When DP-1 = `iis` (or default)
- Question: "Virtual directory name? Default: `{KBName}NetCoreSQL` or custom?"
- Options:
	* `default`: (DEFAULT) Convention `{KBName}{PlatformShort}{DBMSShort}`. Example: KB `ClienteA` + SQL Server = `ClienteANetCoreSQL`, + PostgreSQL = `ClienteANetCorePostgreSQL`
	* `custom`: User-defined. Must be URL-safe
- Impact: Affects the base access URL and the WebApplication name in IIS
- Phase: deploy

### DP-3: stdout Logs Directory
- Trigger: When deploying to IIS (DP-1 = `iis`)
- Question: (Do not ask; ALWAYS create `{TargetPath}/web/logs/` automatically and notify the user)
- Auto-action: `mkdir -p {TargetPath}/web/logs`; without this directory, stdout logging fails silently
- Phase: deploy
