---
name: setup-recipe
description: Fresh GAM-enabled KB recipe; use when activating GAM from scratch, writing the GAM activation properties and credentials, declaring the GAM DataStore, or smoke-testing the GAM login endpoint
---

# GAM KB Setup: Fresh GAM Recipe (GAM-specific steps)
Scope: this file documents ONLY the GAM-specific aspects of setting up a fresh GAM-enabled KB. Anything related to GeneXus mechanics (KB creation, environment file syntax, import/export, build, Virtual Directory) MUST be sourced from the Nexa skill; this file does not duplicate that knowledge

For an EXISTING GAM database, see [Connecting a New KB to an Existing GAM Database (GAM-specific recipe)](connect-existing-gam-db.md)

Related (GAM):
- [GAM KB Setup: Orchestrator (GAM scope only)](domain-kb-setup.md): GAM property scopes, build phases, post-build verification
- [GAM KB Setup: Troubleshooting Catalog](kb-setup-troubleshooting.md): GAM-specific errors during setup

Related (Nexa, DELEGATE):
- KB creation: Nexa skill (`model-knowledge-base.md`, `mcp__genexus__create_knowledge_base` tool)
- Generic environment file semantics (`#Backend`, `#Properties`, DataStore syntax beyond GAM): Nexa skill (`model-environment.md`, `properties-environment-*.md`)
- Import via `gx` (`import_text_to_kb` names format and behavior): Nexa skill
- Build invocation (`build_all`, MSBuild): Nexa skill
- Database creation (`create_or_impact_database`): Nexa skill
- IIS Virtual Directory creation: Nexa skill (or platform documentation; currently a Nexa gap; ask before assuming)

---

## Workflow Overview (GAM lens)
The end-to-end flow has GeneXus-mechanic steps and GAM-specific steps. The GAM skill ONLY owns the GAM-specific ones

GeneXus-mechanic steps (Nexa owns these: consult Nexa for syntax/commands):

- Create the KB (file system + KB metadata DB)
- Open the KB
- Write/edit environment text files using Nexa file-format rules
- Import the configuration files into the KB
- Invoke the build
- Create the application database
- Configure IIS Virtual Directory

GAM-specific steps (this file owns these):

- Write the GAM activation properties and credentials ([GAM Activation Snippet](#gam-activation-snippet))
- Declare BOTH Default and GAM DataStores ([DataStore Declaration: Default + GAM](#datastore-declaration-default--gam))
- Understand the GAM Integrated Security Initialization phase of the build ([Build: GAM Integrated Security Initialization Phase](#build-gam-integrated-security-initialization-phase))
- Grant IIS AppPool perms on the GAM DB ([AppPool Permissions on the GAM Database](#apppool-permissions-on-the-gam-database))
- Smoke-test the GAM login page ([Smoke Test: GAM Login Endpoint](#smoke-test-gam-login-endpoint))

Consult the Nexa skill for any GeneXus-mechanic step before executing it; do NOT inline tool syntax or build commands in this skill

One exception: the GAM property names, values, and placement are authoritative HERE. Nexa does not document them, so an agent sent to Nexa for that format finds nothing, guesses, and then stalls or defers to the IDE. See [GAM Activation Snippet](#gam-activation-snippet)

---

## GAM Activation Snippet
This snippet is AUTHORITATIVE for GAM property names, values, and placement; write it as-is and do not look for the format elsewhere. Only the surrounding mechanics stay delegated: KB creation, import, build, and DB creation are Nexa scope

Write `src/#preferences/<KBName>.kb.gx`, block `#Version`:

```
EnableIntegratedSecurity = "True"
IntegratedSecurityLevel = "Authorization"
LoginObjectForWeb = "GAMExampleLogin"
```

Write `src/#preferences/<EnvName>.env.gx`, block `#Backend`; this file is shared, so it carries no secret:

```
DataStores
{
	Default
	[
		Dbms = 'SQL Server'
	]

	GAM
	[
		Dbms = 'SQL Server'
	]
}
```

Write `src/#preferences/<EnvName>.local.env.gx`; this file is git-ignored, so every secret belongs here:

```
#Properties
	AdministratorUserName = "admin"
	AdministratorUserPassword = "<admin-password>"
	ConnectionUserName = "<KBName>"
	ConnectionUserPassword = "<connection-password>"
#End

#Backend
	DataStores
	{
		Default
		[
			DatabaseName = '<KBName>',
			ServerName = '<host>\\<instance>',
			UseTrustedConnection = 'Yes'
		]

		GAM
		[
			DatabaseName = '<KBName>_GAM',
			ServerName = '<host>\\<instance>',
			UseTrustedConnection = 'Yes'
		]
	}
#End
```

Rules:
- Write every property name in PascalCase, unquoted; a quoted display name with spaces is ignored silently
- Resolve `<EnvName>` from `CurrentEnvironment` in `<KBName>.local.kb.gx`; never assume a single environment
- OMIT `ApplicationId` and `RepositoryId` for a fresh GAM; GeneXus generates both at first build
- Ask the user for both passwords; never reuse a shared or default password
- Replace `'SQL Server'` with the target DBMS value when the environment is not SQL Server
- Use `UserId` and `UserPassword` instead of `UseTrustedConnection` for DBMS authentication

For an existing GAM DB both identifiers become mandatory; see [Connecting a New KB to an Existing GAM Database (GAM-specific recipe)](connect-existing-gam-db.md)

For scoping rules and the names that fail, see [GAM Property Scope and Semantics](domain-kb-setup.md#gam-property-scope-and-semantics) and [Property Names That CAUSE ERRORS](domain-kb-setup.md#property-names-that-cause-errors)

---

## DataStore Declaration: Default + GAM
A GAM-enabled KB requires TWO DataStores; the GAM one is named exactly `GAM`. The generic DataStore syntax and import behavior are Nexa scope. This section documents ONLY the GAM-specific facts

- The Default DataStore is the application DB; naming convention `<KBName>` with no suffix
- The GAM DataStore holds GAM tables such as users, sessions, and tokens; naming convention `<KBName>_GAM`
- Both DataStores typically share the same server, port, and authentication mode
- Declare the `Dbms` value in `<EnvName>.env.gx` and the connection details in `<EnvName>.local.env.gx`
- Keep credentials and server names out of `<EnvName>.env.gx`; that file is shared and versioned

After writing the files, import them via Nexa's `import_text_to_kb` tool; see Nexa for the correct `names` parameter

---

## Build: GAM Integrated Security Initialization Phase
When the build runs (via Nexa's build tool), it includes a GAM-specific phase: "Integrated Security Initialization". This phase:

- Publishes the GAM init project (`GxDeps.csproj` from `Library/GAM/Platforms/…`)
- Generates `<KBPath>/<TargetPath>/Library/GAM/client.exe.config` with the resolved DataStore values
- Connects to the GAM database (`<KBName>_GAM`): fails the build if unreachable
- Retrieves GAM version, registers the application, deploys GAM modules

The most common build failure is in this phase. Look at the log lines just BEFORE "Build canceled" for the actual error (typically a SQL connection failure). See [GAM KB Setup: Troubleshooting Catalog](kb-setup-troubleshooting.md)

CRITICAL: `client.exe.config` is REGENERATED at the start of every build from the KB internal DataStore storage. Manual edits are overwritten. To fix connection issues, fix the source `<EnvName>.local.env.gx` file and re-import via Nexa

---

## AppPool Permissions on the GAM Database
When deploying via IIS with Trusted Connection, the IIS AppPool identity (e.g., `IIS AppPool\DefaultAppPool`) needs `db_owner` (or equivalent) on BOTH databases. The GAM-specific part is the GAM DB:

```sql
USE [<KBName>_GAM];
IF NOT EXISTS (SELECT 1 FROM sys.database_principals WHERE name = 'IIS AppPool\DefaultAppPool')
  CREATE USER [IIS AppPool\DefaultAppPool] FOR LOGIN [IIS AppPool\DefaultAppPool];
ALTER ROLE db_owner ADD MEMBER [IIS AppPool\DefaultAppPool];
```

The `CREATE LOGIN` and the equivalent grant on the Default DB are general GeneXus deployment concerns: see Nexa or platform documentation

Without this grant, GAM endpoints return HTTP 500 with `Login failed for user 'IIS AppPool\DefaultAppPool'` in stdout logs (Error Code 4060)

---

## Smoke Test: GAM Login Endpoint
After the Virtual Directory is configured (see Nexa for that step), the GAM-specific smoke test is the login WebPanel:

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost/<VirtualDir>/gamexamplelogin.aspx
# Expected: 200
```

Verify the response body contains the GAM login form (`vUSERNAME`, `vUSERPASSWORD`, `LOGIN` button). A 200 response with non-GAM content means the wrong page is being served

Other GAM-specific endpoints to test (depending on configured features):

- `/<VirtualDir>/oauth/access_token`: REST IDP token endpoint
- `/<VirtualDir>/.well-known/openid-configuration`: OIDC discovery (only if OIDC is configured)

For non-GAM-specific HTTP failures (404 = Virtual Directory missing, 500 = AppPool/DB issue), consult Nexa or platform docs

---

## Decision Points (GAM scope)
See [Decision Points](domain-kb-setup.md#decision-points) for the full Decision Points list (GAM scope vs Nexa scope). At a minimum, this recipe needs:

- DP-GAM-1: Fresh GAM or existing GAM DB? (this file = `fresh`; for existing, switch to [Connecting a New KB to an Existing GAM Database (GAM-specific recipe)](connect-existing-gam-db.md))
- DP-GAM-2: Admin credentials (default user `admin`; ask the user for the password)
- DP-GAM-3: Connection user (default user `<KBName>`; ask the user for the password)

All non-GAM DPs (platform, DBMS, KB name, server, build method, virtual directory name) belong to Nexa: do NOT ask them from this skill
