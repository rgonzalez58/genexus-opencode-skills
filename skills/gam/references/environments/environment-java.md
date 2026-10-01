---
name: environment-java
description: Java environment; Tomcat, client.cfg, Gradle, JDBC, WAR deployment
---

# GeneXus Environment: Java

## Identity
- Generator (text format): `Java`
- Language (text format): `Java`
- Generator ID: 12
- Model ID: 32
- GXSPC directory: GXSPC032
- Build system: Gradle (GeneXusSecurity.gradle)
- VER file pattern: `GJS<build>.VER`
- PRO file pattern: `GX_PRO12.032`

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
					Generator = 'Java'
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
		Language = "Java"
		DataSource = "SQL Server"
		TargetPath = "{TargetDir}"
		Name = "{EnvironmentName}"
	#End
}
```

Replace `'SQL Server'` with `'MySQL'` for MySQL environments

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

## Runtime Config: `client.cfg` (INI)
- Located: `{TargetPath}/web/client.cfg`
- Format: INI with `[Client]`, `[namespace]`, `[namespace|DataStore]` sections
- `MODEL_NUM = 32`, `GENERATOR_NUM = 12`

### Key Sections
```ini
[Client]
MODEL_NUM= 32
GENERATOR_NUM= 12
PACKAGE=genexus.programs
EnableIntegratedSecurity=0
IntegratedSecurityLoginWeb=gamexamplelogin

[genexus.programs]
DataSource1=GAM
DataSource2=DEFAULT

[genexus.programs|GAM]
CS_DBNAME={db_name}
DBMS=sqlserver
JDBC_DRIVER=com.microsoft.sqlserver.jdbc.SQLServerDriver
DB_URL=jdbc:sqlserver://<server>:<port>;databaseName=<db>;encrypt=true;trustServerCertificate=true
USER_ID={encrypted_base64}
USER_PASSWORD={encrypted_base64}
```

### JDBC Connection Strings per DBMS
- SQL Server
	* JDBC Driver: `com.microsoft.sqlserver.jdbc.SQLServerDriver`
	* URL Pattern: `jdbc:sqlserver://<server>:<port>;databaseName=<db>`
- MySQL
	* JDBC Driver: `com.mysql.cj.jdbc.Driver`
	* URL Pattern: `jdbc:mysql://<server>:<port>/<db>`
- PostgreSQL
	* JDBC Driver: `org.postgresql.Driver`
	* URL Pattern: `jdbc:postgresql://<server>:<port>/<db>`

### Credentials
Java environments encrypt `USER_ID` and `USER_PASSWORD` with base64-encoded symmetric encryption. Java always uses explicit credentials; Windows Auth requires special JDBC driver configuration (`integratedSecurity=true` + `sqljdbc_auth.dll`)

---

## GAM Init: `client.cfg` (Library)
- Located: `{TargetPath}/Library/GAM/client.cfg`
- Same INI format as runtime `client.cfg`
- Also has `reorg.cfg` for reorganization config

### InputParms.json (Full IdeConnection)
```json
{
	"Applications": null,
	"FilesPath": "{KBPath}/GXSPC032/Security",
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
MSBuild orchestrates the GeneXus build; Gradle compiles the Java code:
```bash
dotnet msbuild BuildServer.msbuild \
	-t:RebuildAndDeploy \
	"-p:GXInstall={GXInstallPath}" \
	"-p:KbLocation={KBPath}" \
	"-p:ActiveVersion={VersionName}" \
	"-p:ActiveEnvironment={EnvironmentName}" \
	-v:normal
```

### Unique Build/Web Files
- `GeneXusSecurity.gradle` in web/
- `ExternalObjects.jar` in web/
- `GAM_Backend/` directory in web/
- `web.xml` in web/ (Jakarta EE servlet mappings)

---

## Deployment: Application Server

### Tomcat
- Deploy web/ as WAR or configure Tomcat context pointing to web/
- Requires Tomcat 10.x+ (Jakarta EE namespace)
- Default port: 8080

### URL Pattern
```
http://localhost:8080/<context>/gamexamplelogin
http://localhost:8080/<context>/oauth/access_token
http://localhost:8080/<context>/.well-known/openid-configuration
```

Key difference: Java does NOT use `.aspx` extension

---

## Troubleshooting

### Tomcat version mismatch
GeneXus uses Jakarta EE: requires Tomcat 10.x+. Older Tomcat (javax.servlet) will fail

### JDBC driver not found
Ensure SQL Server JDBC driver JAR is in Tomcat lib/ or webapp classpath

### No Windows Auth by default
Java JDBC needs `integratedSecurity=true` + `authenticationScheme=NativeAuthentication` + `sqljdbc_auth.dll` on PATH. Simpler to use SQL Auth

### client.cfg encoding issues
INI format: watch for spaces in values and section names matching the package namespace

---

## Decision Points
These decisions apply when this environment skill is active (user chose Java). Present applicable ones before proceeding

### DP-1: Application Server
- Trigger: Before deploying the Java application
- Question: "Deploy on `Tomcat` (default) or another application server (WildFly, WebLogic, etc.)?"
- Options:
	* `tomcat`: (DEFAULT) Apache Tomcat 10.x+ (Jakarta EE). Requires JDBC driver in classpath
	* `other`: Specify which. Must support Jakarta EE (Servlet 5.0+)
- Impact: Changes the deploy method (WAR vs context), configuration paths, and troubleshooting
- Phase: deploy

### DP-2: Context Name (equivalent to Virtual Directory)
- Trigger: Before deploying a WAR or configuring Tomcat context
- Question: "Context name? Default: `{KBName}JavaSQL` or custom?"
- Options:
	* `default`: (DEFAULT) Convention `{KBName}{PlatformShort}{DBMSShort}`. Example: KB `ClienteA` + SQL Server = `ClienteAJavaSQL`, + MySQL = `ClienteAJavaMySQL`, + SAP HANA = `ClienteAJavaSAPHana`
	* `custom`: User-defined. Must be URL-safe
- Impact: Affects the base URL: `http://localhost:8080/<context>/`
- Phase: deploy

### DP-3: Database Authentication
- Trigger: When configuring JDBC connections
- Question: "SQL authentication (user/password, default for Java) or Windows Auth (requires special config)?"
- Options:
	* `sql_auth`: (DEFAULT for Java) Explicit USER_ID/USER_PASSWORD. Simpler, works on all OS
	* `windows_auth`: Requires `integratedSecurity=true` + `sqljdbc_auth.dll` on PATH. Windows only, SQL Server only
- Impact: Java does NOT have Windows Auth by default like .NET. SQL Auth is the natural option
- Phase: design
