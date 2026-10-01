---
name: environment-netframework
description: .NET Framework environment; IIS, web.config, virtual directories
---

# GeneXus Environment: .NET Framework

## Identity
- Generator (text format): `C#`
- Language (text format): `C#`
- Generator ID: 15
- Model ID: 31
- GXSPC directory: GXSPC031
- Build targets: DotNetBaseProject.targets
- VER file pattern: `GNetWebB<build>.VER`
- PRO file pattern: `GX_PRO15.031`

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
					Generator = 'C#'
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
		Language = "C#"
		DataSource = "SQL Server"
		TargetPath = "{TargetDir}"
		Name = "{EnvironmentName}"
		"Virtual Directory Name" = "{VirtualDirName}"
	#End
}
```

Replace `'SQL Server'` with `'MySQL'` or `'PostgreSQL'` for other DBMS

### Local (`{EnvName}.local.env.gx`)
```
Environment {EnvironmentName}
{
	#Backend
		DataStores
		{
			Default
				[
					DatabaseName = '{AppDB}',
					ServerName = '{ServerHost}',
					UserId = '',
					UserPassword = ''
				]
			GAM
				[
					DatabaseName = '{AppDB}_GAM',
					ServerName = '{ServerHost}',
					UserId = '',
					UserPassword = ''
				]
		}
	#End
}
```

Trusted Connection: Set `UseTrustedConnection = 'Yes'` in the `{EnvName}.local.env.gx` file with empty `UserId`/`UserPassword`. After writing, import via `import_text_to_kb` with `names: ["environment:*"]`

### GAM Environment-scope Properties
When this environment runs with GAM, add to the `#Properties` block of `{EnvName}.local.env.gx`; these carry secrets and never belong in the shared `{EnvName}.env.gx`:
- `AdministratorUserName`, `AdministratorUserPassword`: GAM init credentials
- `ConnectionUserName`, `ConnectionUserPassword`: Repository connection credentials
- `RepositoryId`: ONLY when connecting to an existing GAM DB

Write every name in PascalCase, unquoted; a quoted display name with spaces is ignored silently

See [GAM Activation Snippet](../kb-setup/setup-recipe.md#gam-activation-snippet) for the exact text, and [Recipe Steps](../kb-setup/connect-existing-gam-db.md#recipe-steps) for the existing-GAM recipe

---

## Runtime Config: `web.config`
- Located: `{TargetPath}/web/web.config`
- Format: ASP.NET XML configuration
- Connection values encrypted (base64): Datasource, User, Password, Schema
- DB name (`Connection-<store>-DB`) in plain text
- No `appsettings.json`: everything in web.config
- Has `<system.web>` with `targetFramework="4.6.2"` (or current)
- Has `<system.serviceModel>` for WCF services

### Key AppSettings
```xml
<add key="EnableIntegratedSecurity" value="1" />
<add key="IntegratedSecurityLoginWeb" value="gamexamplelogin" />
<add key="IntegratedSecurityNotAuthorizedWeb" value="gam_notauthorized" />
```

---

## GAM Init: `client.exe.config`
- Located: `{TargetPath}/Library/GAM/client.exe.config`
- Format: XML with `<datastores>` section
- Values in plain text (NOT encrypted: unlike web.config)
- `MODEL_NUM = 31`, `GENERATOR_NUM = 15`
- Regenerated from DataStore blob at every build start

### InputParms.json (Full IdeConnection)
```json
{
	"Applications": null,
	"FilesPath": "{KBPath}/GXSPC031/Security",
	"IdeConnection": {
		"ConnPassword": "{conn_password}",
		"ConnUser": "{conn_user}",
		"Repository": "{repository_guid}",
		"UserName": "{admin_user}",
		"UserPassword": "{admin_password}"
	},
	"RepositoryGUID": null
}
```

---

## Build
```bash
dotnet msbuild BuildServer.msbuild \
	-t:RebuildAndDeploy \
	"-p:GXInstall={GXInstallPath}" \
	"-p:KbLocation={KBPath}" \
	"-p:ActiveVersion={VersionName}" \
	"-p:ActiveEnvironment={EnvironmentName}" \
	-v:normal
```

### Unique Build Files
- `DotNetBaseProject.targets` in build/
- `LastBuild.sln` in build/

---

## Deployment: IIS Virtual Directory (DEFAULT; always do this)
Virtual Directory naming convention: `{KBName}{EnvironmentSuffix}`
Where EnvironmentSuffix matches the DBMS combination: `NetSQL`, `NetMySQL`, etc

### Automatic
GeneXus generates `VirtualDir.exe` in web/ during build. Run it to create the IIS virtual directory
- `VirtualDirCreated` flag file = success
- Uses the `"Virtual Directory Name"` property from environment text format

### Manual IIS Setup
- Open IIS Manager
- Default Web Site > Add Application
- Alias: `{KBName}{EnvSuffix}` (e.g., `MyKBNetSQL`)
- Physical Path: `{KBPath}/{TargetPath}/web`
- Application Pool: .NET Framework 4.x, Integrated pipeline

### Key Files in web/
- `GXExec.exe`: GeneXus runtime executor
- `VirtualDir.exe`: IIS Virtual Directory creator
- `GXDataInitialization.exe` in bin/: data initialization
- `web.config`: runtime configuration

### URL Pattern
```
http://localhost/{VirtualDir}/gamexamplelogin.aspx
http://localhost/{VirtualDir}/oauth/access_token
http://localhost/{VirtualDir}/.well-known/openid-configuration
```

---

## CRITICAL: IIS AppPool SQL Server Permissions (Trusted Connection)
MUST DO after database creation when using Integrated Security. Same issue as .NET Core

The IIS AppPool identity (`IIS AppPool\DefaultAppPool`) needs explicit SQL Server login + db_owner on BOTH Default and GAM databases. Without this: HTTP 500 with `Login failed for user 'IIS AppPool\DefaultAppPool'`

```sql
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

---

## Session Cookies
- `ASP.NET_SessionId`: ASP.NET Framework server-side session identifier (InProcess)
- `GAMSessionGUID`: GAM session identifier (persists across requests)

---

## Hosting Model
- IIS runs InProcess via `w3wp.exe` (classic ASP.NET pipeline)
- GAM traces write to `<KB>/NetModel/web/bin/client.log`
- Recycle the AppPool to force GAM configuration reload (for example, after updating trace switches: see [Cache Expiry Requirement](../debugging/common-debugging.md#cache-expiry-requirement)):

```powershell
& "$env:windir\system32\inetsrv\appcmd.exe" recycle apppool /apppool.name:"DefaultAppPool"
```

---

## Troubleshooting

### HTTP 500: Login failed for 'IIS AppPool\DefaultAppPool'
Number 1 most common IIS deployment issue. See "IIS AppPool SQL Server Permissions" section above

### IIS not installed
VirtualDir.exe fails silently. Install IIS with ASP.NET 4.x feature enabled

### Wrong Application Pool
Must be .NET Framework 4.x Integrated pipeline, NOT .NET Core

### web.config locked by IIS
Stop IIS (`iisreset /stop`) before cleaning web/ directory

### Cannot edit connection strings in web.config
Values are encrypted. Fix the `{EnvName}.local.env.gx` file and reimport via `import_text_to_kb` instead

### DB name mismatch (GX_KB_ prefix)
Same as .NET Core: verify `Connection-Default-DB` in web.config matches actual SQL database name

---

## Decision Points
These decisions apply when this environment skill is active (user chose .NET Framework). Present applicable ones before proceeding

### DP-1: Virtual Directory Name
- Trigger: Before deployment (build generates VirtualDir.exe)
- Question: "Virtual directory name? Default: `{KBName}NetSQL` or custom?"
- Options:
	* `default`: (DEFAULT) Convention `{KBName}{PlatformShort}{DBMSShort}`. Example: KB `ClienteA` + SQL Server = `ClienteANetSQL`, + MySQL = `ClienteANetMySQL`
	* `custom`: User-defined. Set in property `"Virtual Directory Name"` of the environment text format
- Impact: Affects the base URL and the value VirtualDir.exe uses to create the virtual directory in IIS
- Phase: deploy

### DP-2: Virtual Directory Creation Method
- Trigger: After build completes
- Question: "Create virtual directory with `VirtualDir.exe` (automatic) or `manual` via IIS Manager?"
- Options:
	* `auto`: (DEFAULT) Run `VirtualDir.exe` generated in web/. Creates the virtual directory automatically
	* `manual`: Configure manually in IIS Manager (useful if VirtualDir.exe fails silently or IIS is not installed)
- Impact: Both achieve the same result. Manual is fallback if auto fails
- Phase: deploy
