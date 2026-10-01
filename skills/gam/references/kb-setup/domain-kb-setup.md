---
name: domain-kb-setup
description: GAM KB setup orchestrator; use when deciding the scope of a GAM property, resolving which model file holds it, or interpreting a GAM build phase, post-build artifact, or setup Decision Point
---

# GAM KB Setup: Orchestrator (GAM scope only)
Scope: this file documents ONLY the GAM-specific aspects of KB setup. Anything about GeneXus mechanics (file formats, environment file syntax, import/export semantics, build invocation, IIS Virtual Directory) is NOT in this file; consult the Nexa skill at runtime

The GAM-specific concerns this file covers:
- Which GAM properties exist and at which scope (Version vs Environment): [GAM Property Scope and Semantics](#gam-property-scope-and-semantics)
- How `ApplicationId` and `RepositoryId` are generated, auto versus manual: [How ApplicationId and RepositoryId Are Generated](#how-applicationid-and-repositoryid-are-generated)
- Property names that CAUSE ERRORS and the correct alternatives: [Property Names That CAUSE ERRORS](#property-names-that-cause-errors)
- The GAM-specific phases inside the build: [Build; GAM-specific Phases](#build-gam-specific-phases)
- The GAM-specific post-build artifacts and verifications: [Post-Build; GAM Artifacts and Verification](#post-build-gam-artifacts-and-verification)
- GAM Decision Points (and what is delegated to Nexa): [Decision Points](#decision-points)

Related files (GAM):
- [GAM KB Setup: Fresh GAM Recipe (GAM-specific steps)](setup-recipe.md): fresh GAM step-by-step (GAM steps only; KB/build/deploy via Nexa)
- [Connecting a New KB to an Existing GAM Database (GAM-specific recipe)](connect-existing-gam-db.md): existing-GAM-DB recipe
- [GAM KB Setup: Troubleshooting Catalog](kb-setup-troubleshooting.md): GAM-specific build errors
- `init/` sub-folder: programmatic GAM entity initialization
- [GAM Authentication Configuration Reference](../authentication/external-providers/common/domain-auth-config.md)

Related (Nexa, DELEGATE):
- KB creation, generic environment file semantics, DataStore syntax, import behavior, build invocation, virtual directory creation: see Nexa skill (`model-knowledge-base.md`, `model-environment.md`, `properties-environment-*.md`)
- GAM property names, values, and placement are NOT delegated; they are authoritative in [GAM Activation Snippet](setup-recipe.md#gam-activation-snippet)

---

## KB Files GAM Touches
GAM properties live in four model files under `src/#preferences/`; these names are authoritative, never inferred:
- `<KBName>.kb.gx`: Block `#Version`; GAM activation, level, login object, `ApplicationId`
- `<KBName>.local.kb.gx`: Block `#Properties`; `CurrentEnvironment` resolves which environment to write
- `<EnvName>.env.gx`: Block `#Backend`; GAM DataStore declaration with its `Dbms` value
- `<EnvName>.local.env.gx`: Blocks `#Properties` and `#Backend`; GAM credentials, `RepositoryId`, GAM DB connection

Rules:
- Forbid a separate Version file; Version properties live inside `#Version` of `<KBName>.kb.gx`
- Read the version name from the `Name` property inside `#Version`; never assume it
- Resolve `<EnvName>` from `CurrentEnvironment`; a KB can hold many environments and the wrong one fails silently
- Write every secret only in `<EnvName>.local.env.gx`; that file is git-ignored
- Forbid abbreviating a model file name as `.local.gx` or `.main.gx`
- Forbid the retired names `*.version.main.gx`, `*.environment.main.gx`, `*.environment.local.gx`

Read the exact text to write from [GAM Activation Snippet](setup-recipe.md#gam-activation-snippet)

Section semantics, import workflow, and generic DataStore syntax are Nexa scope; read them from Nexa

---

## GAM Property Scope and Semantics
Write every property name in PascalCase, unquoted, without spaces. A quoted display name with spaces is an IDE label; it is invalid in a `.gx` file and is ignored silently

### Version-scope (file: `<KBName>.kb.gx`, `#Version` block)
- `EnableIntegratedSecurity`: Activates GAM in this KB; value `"True"` as a quoted string
- `IntegratedSecurityLevel`: Value `"Authentication"` or `"Authorization"`; maps internally to SecurityMedium and SecurityHigh
- `ApplicationId`: Free-form string identifying the application within GAM; see [ApplicationId at Version scope](#applicationid-at-version-scope) for generation rules
- `LoginObjectForWeb`: Login WebPanel name; value `"GAMExampleLogin"` by default or a custom WebPanel

### Environment-scope (file: `<EnvName>.local.env.gx`, `#Properties` block)
These properties select which GAM Repository the build targets and which credentials GAM init uses; they carry secrets, so they belong in the git-ignored local override and never in `<EnvName>.env.gx`
- `AdministratorUserName`: Admin account GAM init creates or uses; value `"admin"` by default
- `AdministratorUserPassword`: Admin password; ask the user for the value, mandatory for an existing GAM
- `ConnectionUserName`: Repository connection account; value `"<KBName>"` by default
- `ConnectionUserPassword`: Connection user password; ask the user for the value, mandatory for an existing GAM
- `RepositoryId`: Repository GUID with dashes; see [RepositoryId at Environment scope](#repositoryid-at-environment-scope), set only for an existing GAM DB

### Property Name Across Layers
The same concept carries a different name in each layer; never carry one layer's name into another:
- `.gx` text format: `RepositoryId`
- KB metadata store: `IntegratedSecurityRepositoryId`
- GAM database: attribute `RepGUID` on table `gam.Repository`, keyed by `RepId`

The GAM database schema is always `gam`; it is hardcoded and never configurable

Remaining metadata identifiers, for reading a diagnostic query result only:
- `IntegratedSecurityAdministratorName` maps to `AdministratorUserName`
- `IntegratedSecurityAdministratorPassword` maps to `AdministratorUserPassword`
- `IntegratedSecurityConnectionUserName` maps to `ConnectionUserName`
- `IntegratedSecurityConnectionPassword` maps to `ConnectionUserPassword`

### Wrong-scope Behavior
- Place `ApplicationId` at Version scope; wrong scope is ignored silently
- Place `RepositoryId` and every credential at Environment scope; wrong scope is ignored silently

---

## How `ApplicationId` and `RepositoryId` Are Generated
These two identifiers bind the KB to a specific Application and Repository in the GAM database. Both matter only when targeting an EXISTING GAM database; for a fresh GAM, GeneXus fills them automatically and setting them manually is wrong. Getting this backwards is the most common cause of `Repository not found (GAM2)`

### `ApplicationId` at Version scope
- Type: Free-form string; not always a GUID despite the DB attribute name
- Fresh GAM: OMIT the property. GeneXus auto-generates it via `IntegratedSecurityApplictationIdDefaultResolver`, which activates whenever `EnableIntegratedSecurity = "True"` is present
- Existing GAM: SET the property to `AppGUID` queried from `gam.Application` for the target Repository. The user MUST provide it

### `RepositoryId` at Environment scope
- Type: GUID with dashes
- Fresh GAM: OMIT the property. GeneXus generates a random GUID at first build, writes it into the KB metadata, and registers it as the Repository during GAM initialization
- Existing GAM: SET the property to `RepGUID` queried from `gam.Repository` where `RepId = 2`; that row is the working Repository
- Existing GAM failure: Without it, the generated random GUID matches no Repository in the existing DB and the build fails at "Creating connection.gam file…" with `Repository not found (GAM2)`
- Visibility: HIDDEN in the IDE unless `EnableIntegratedSecurity=true` AND `ModelType != Design`; `FlagExport=false` blocks text export but not text import, so text import is the canonical way to set it

Query the existing value with:

```sql
SELECT RepGUID FROM gam.Repository WHERE RepId = 2;
```

### Mismatch is silent
Setting `RepositoryId` at Version scope or `ApplicationId` at Environment scope produces no warning; the property is ignored, the build proceeds with the auto-generated value, and the failure surfaces later as a GAM2 error

---

## Property Names That CAUSE ERRORS
These names appear in older GAM documentation or generator code and fail in the current KB text format. Use the names from [GAM Property Scope and Semantics](#gam-property-scope-and-semantics) instead

- `GAMAdministratorUser`: Fails with `NullReferenceException` at import; use `AdministratorUserName` at Environment scope
- `GAMAdministratorPassword`: Fails with `NullReferenceException` at import; use `AdministratorUserPassword` at Environment scope
- Any quoted display name with spaces such as `"Administrator User Name"` or `"Login Object for Web"`: Ignored silently, then the build fails in GAM init with an unrelated message
- `TrustedConnection` in environment text: Fails with `NullReferenceException`; use `UseTrustedConnection`, a Nexa-defined property name
- `CS_TRUSTED` in the internal storage blob: Ignored silently; use `TRUSTED_CONNECTION`, managed by Nexa import
- `ApplicationID` with an uppercase `ID`: Ignored silently; use `ApplicationId` at Version scope

---

## Build: GAM-specific Phases
The Nexa skill owns the build invocation (`gx build-all`, MSBuild). The build pipeline includes GAM-specific phases that this skill owns

### Integrated Security Initialization (PHASE 2 of build)
This phase runs BEFORE code generation. It:

- Publishes the GAM init project (`Library/GAM/Platforms/<Platform>/GxDeps.csproj`) to its install path
- Generates `<KBPath>/<TargetPath>/Library/GAM/client.exe.config` from the resolved KB DataStore configuration (see ["client.exe.config" Lifecycle](#clientexeconfig-lifecycle))
- Connects to the GAM database: fails the build with "Build canceled" if unreachable
- Retrieves GAM version, registers the application, deploys GAM modules

Most build failures happen in this phase. When "Build canceled" appears in the log, look at the lines just BEFORE it for the actual error (typically a SQL connection failure to the GAM DB)

### `client.exe.config` Lifecycle
- Path: `<KBPath>/<TargetPath>/Library/GAM/client.exe.config`
- REGENERATED at the start of every build from the KB internal DataStore storage, which Nexa populates from `<EnvName>.local.env.gx`
- Manual edits are overwritten on every build
- If the regenerated config has incorrect connection details, fix the source `<EnvName>.local.env.gx` and re-import; the import itself is Nexa scope

DataStore property → config key mapping (for diagnosing what GAM init received):

- `DatabaseName` → `Connection-<store>-DB` (example: `<KBName>_GAM`)
- `ServerName` → `Connection-<store>-Datasource` (example: `localhost`)
- `UseTrustedConnection=Yes` → `Connection-<store>-Opts` = `;Integrated Security=Yes;`
- `UserId` → `Connection-<store>-User`
- `UserPassword` → `Connection-<store>-Password`

When the value shows as a placeholder (`{UserName}`, `Datasource = "sql"`, hash-like DB name like `Idf02bbf10843…`) → the import did NOT update the internal storage; the build is using the template config. Fix via the Nexa import workflow

### Other GAM-related build phases (high-level)
- Deploy GAM: copies GAM files to `web/`
- GAM Applications Registration: registers the application in the GAM DB
- GAM Permissions Creation: registers configured permissions

---

## Post-Build: GAM Artifacts and Verification

### GAM-specific files to verify
Under `<TargetPath>/web/`:

- `application.gam`: must contain `<Id><KBName></Id>` confirming the application is registered
- `connection.gam`: must contain a `<Key>` GUID (the GAM connection encryption key used at runtime)
- `appsettings.json` (or `web.config` for .NET FW; `client.cfg` for Java): `IntegratedSecurityLoginWeb` = `gamexamplelogin`; `Connection-GAM-Opts` correct (`;Integrated Security=yes;` for Trusted Connection)
- `gamexamplelogin.cs` (or `.java`): login WebPanel was generated, proving UI compilation succeeded

### GAM Login Smoke Test
```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost/<VirtualDir>/gamexamplelogin.aspx
# Expected: 200
```

The Virtual Directory creation itself is Nexa scope. This curl is the GAM-side check

### IIS AppPool Permissions on the GAM DB
When deploying via IIS with Trusted Connection, the AppPool identity needs `db_owner` on BOTH databases. The GAM-specific part is the GAM DB: see [AppPool Permissions on the GAM Database](setup-recipe.md#apppool-permissions-on-the-gam-database) for the SQL grant. Default DB perms are general GeneXus deployment

Without GAM DB perms: HTTP 500 with `Login failed for user 'IIS AppPool\DefaultAppPool'` (Error Code 4060) when the app tries to access GAM tables

---

## Decision Points
Decision Points are split between GAM scope (asked by THIS skill) and Nexa / Infra scope (asked by Nexa or via the platform layer). This skill ONLY asks GAM-specific questions

### GAM scope (this skill asks)

#### DP-GAM-1: Fresh or Existing GAM Database?
- Trigger: user asks to set up a KB with GAM
- Question: "Fresh GAM (DB from scratch) or connecting to an existing GAM database?"
- Default: `fresh`
- Impact:
	* `fresh`: use [GAM KB Setup; Fresh GAM Recipe (GAM-specific steps)](setup-recipe.md). GAM init creates admin user and Repository. Both DBs are created. Application ID and Repository ID auto-generated
	* `existing`: use [Connecting a New KB to an Existing GAM Database (GAM-specific recipe)](connect-existing-gam-db.md). User MUST provide admin password and connection user password (cannot be recovered from DB). Application ID and Repository ID MUST be set to the existing values
- Sub-flow when `existing`: ask whether the user provides credentials or wants to configure them privately

#### DP-GAM-2: Admin Credentials
- Trigger: any GAM activation
- Question: "GAM admin credentials? Default user: `admin`, password: ask the user"
- Defaults: `AdministratorUserName = "admin"` and `AdministratorUserPassword = "<admin-password>"`; ask the user for the value
- Note: never reuse a shared/default password in production

#### DP-GAM-3: Connection User Credentials
- Trigger: GAM activation (always); for existing GAM DB the case-sensitivity matters strictly
- Question: "Connection user? Default user: `<KBName>`, password: ask the user"
- Defaults for fresh GAM: `ConnectionUserName = "<KBName>"` and `ConnectionUserPassword = "<connection-password>"`; ask the user for the value
- Defaults (existing GAM): `<original_kb_name>` / `<connection-password>` (ask the user), where `<original_kb_name>` is the KB that ORIGINALLY populated the GAM DB
- For existing GAM: case-sensitive (stored encrypted in `gam.SysConnectionConfig`). On `Repository connection failed for user X` → name/case is wrong; verify against another working KB pointing to the same GAM DB

### Nexa / Infra scope (DELEGATED: this skill does NOT ask)
The following are Nexa or platform concerns:

- KB name, KB location
- Platform / Generator (.NET FW, .NET Core, Java)
- DBMS (SQL Server, PostgreSQL, MySQL, etc.)
- Server hostname, instance, port
- Database authentication mode (Trusted Connection vs SQL Auth)
- Build method (`gx` vs MSBuild)
- Virtual Directory name and creation method
- Target path, output folder structure

Exception: if a GAM concern critically requires an infra detail (e.g., generator type to locate `connection.gam` correctly), ask as a targeted follow-up, not in the initial DP batch

### Decision Point Grouping (presentation)
When asking GAM DPs, present ONLY GAM-scope questions. Example for fresh GAM:

```
About GAM:
1. Fresh GAM (DB from scratch) or existing GAM DB? (default: fresh)
2. Admin credentials? (default user: admin; ask the user for the password)
3. Connection user? (default user: <KBName>; ask the user for the password)
```

Infra questions (platform, server, virtual dir) are presented by Nexa or the platform layer; not by this skill

Rule: when the user does NOT provide GAM credentials in the initial prompt, ASK; always propose the default convention as the suggested option. Do not silently invent custom values
