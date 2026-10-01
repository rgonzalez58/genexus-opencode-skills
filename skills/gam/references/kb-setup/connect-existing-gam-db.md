---
name: connect-existing-gam-db
description: Existing GAM database recipe; use when a GAM DB already exists and a KB must join it, when querying the GAM DB for the identifiers to configure, or when diagnosing a repository mismatch at build time
---

# Connecting a New KB to an Existing GAM Database (GAM-specific recipe)
Scope: this file documents ONLY the GAM-specific aspects of pointing a new KB to an existing GAM database. The KB creation, environment file format, import, and build steps are NEXA scope; consult Nexa at runtime, do not duplicate that knowledge here

Common scenario: multiple applications share one GAM, or an existing GAM was set up by a prior project and a new KB joins it

Related (GAM):
- [GAM KB Setup: Orchestrator (GAM scope only)](domain-kb-setup.md): GAM property scopes and DPs
- [GAM KB Setup: Fresh GAM Recipe (GAM-specific steps)](setup-recipe.md): fresh GAM (the baseline this recipe diverges from)
- [GAM KB Setup: Troubleshooting Catalog](kb-setup-troubleshooting.md): including the GAM2 "Repository not found" error

Related (Nexa, DELEGATE):
- KB creation, open, environment file format, DataStore syntax, import, build, DB creation: see Nexa skill

---

## The Database is the Source of Truth
When GAM already exists, the existing database is the source of truth: defaults do NOT apply. Before configuring, identify what can be queried vs what the user MUST provide

- `RepGUID`: can be queried YES via `SELECT RepGUID FROM gam.Repository WHERE RepId = 2`; user must provide YES, ask for it; feeds the `RepositoryId` property
- `AppGUID`: can be queried YES via `SELECT AppGUID FROM gam.Application WHERE RepId = 2`; user must provide YES, ask for it; feeds the `ApplicationId` property. NOTE: despite the attribute name, this is a free-form string identifier and not always a GUID; use whatever value is stored
- Admin username: can be queried YES via `SELECT UserName FROM gam.User WHERE UserName LIKE '%admin%'`; user must provide YES, ask for it, name only
- Admin password: cannot be queried, stored hashed; user must provide it, mandatory
- Connection username: can be queried YES via `SELECT RepConUser FROM gam.RepositoryConnection`; user must provide YES, ask for it, name only and case-sensitive
- Connection user password: cannot be queried, stored encrypted; user must provide it, mandatory
- Connection Key GUID: can be queried YES via `SELECT SysConnCfgKey FROM gam.SysConnectionConfig`; user must provide YES, ask for it, used only for the post-build `connection.gam` fallback; see "Fallback; Post-Build connection.gam Replacement" below

CRITICAL: the value used by GAM at runtime is `RepGUID`, the Repository GUID, NOT the numeric `RepId`. Always query and configure the GUID. The GAM database schema is always `gam`; it is hardcoded and never configurable

---

## Skill Limitation: Two Irrecoverable Secrets
The skill CANNOT determine: **admin password** and **connection user password**. These two values MUST be provided by the user. There is no workaround; the GAM database stores them hashed/encrypted respectively

---

## Step 0: Ask About Credential Privacy
Before doing anything else, ask:

> "You already have a GAM database. Do you want to give me the credentials (admin password and connection user password) so I configure everything, or are they private and you prefer to configure those steps manually?"

Two paths:
- **Claude configures** → ask for both passwords + the queryable values from "The Database is the Source of Truth" above, configure all files
- **User configures** → tell the user EXACTLY which files to edit and where each value goes; Claude does everything else

---

## How `ApplicationId` and `RepositoryId` Are Generated
These two identifiers BIND a KB to a specific Application and Repository in the GAM database. Both are only set when joining an existing GAM; a fresh GAM fills them automatically. Getting this wrong is the most common cause of GAM2 errors

Write both names in PascalCase, unquoted; a quoted display name with spaces is an IDE label and is ignored silently

### `ApplicationId` at Version scope
- Where it lives: `<KBName>.kb.gx` `#Version` block
- Type: free-form string, despite the `AppGUID` attribute name in the DB
- Fresh GAM: OMIT the property. GeneXus auto-generates via `IntegratedSecurityApplictationIdDefaultResolver`, which activates whenever `EnableIntegratedSecurity = "True"`. Do NOT set it manually
- Existing GAM: SET the property to `AppGUID` queried from `gam.Application` for the application you are joining. This is a value the user MUST provide; see "The Database is the Source of Truth" above

### `RepositoryId` at Environment scope
- Where it lives: `<EnvName>.local.env.gx` `#Properties` block; at Version scope it is ignored silently
- Type: GUID with dashes
- Source attribute: `RepGUID` on table `gam.Repository`, the row where `RepId = 2`
- Fresh GAM: OMIT the property. GeneXus generates a RANDOM GUID at first build and writes it into the KB metadata. The build registers this new GUID as the Repository in the GAM database
- Existing GAM: SET the property to the existing `RepGUID`. Without this, the generated random GUID matches no Repository in the existing DB and the build fails at "Creating connection.gam file…" with **`Repository not found (GAM2)`**
- Visibility quirk: `RepositoryId` is HIDDEN in the IDE unless `EnableIntegratedSecurity=true` AND `ModelType != Design`. Its `FlagExport=false`, so it does NOT round-trip via text export, but DOES import via text. Text import is the canonical way to set it
- KB metadata identifier, for diagnostic queries only, never in a `.gx` file: `IntegratedSecurityRepositoryId`

### Wrong-scope behavior
- `ApplicationId` at Environment scope → silently ignored
- `RepositoryId` at Version scope → silently ignored
- Mismatch → silent: the build proceeds with the auto-generated random GUID and fails later with GAM2

---

## What Changes vs Fresh GAM
Compared to [GAM KB Setup: Fresh GAM Recipe (GAM-specific steps)](setup-recipe.md), the existing-GAM recipe differs in:

- `ApplicationId` at Version scope: fresh OMIT, auto-generated; existing SET to `AppGUID` from `gam.Application`
- `RepositoryId` at Environment scope: fresh OMIT, auto-generated; existing REQUIRED, set to `RepGUID` from `gam.Repository`
- Admin password at Environment scope: fresh asks the user for a new value; existing reuses the existing admin password, which the user provides
- Connection user and password at Environment scope: fresh uses KB-name based defaults; existing reuses the existing values, case-sensitive
- GAM DB name, in the `GAM` DataStore of `<EnvName>.local.env.gx`: fresh `<KBName>_GAM`; existing keeps its current GAM DB name
- Create DB step: fresh creates both Default and GAM; existing creates ONLY Default, since GAM is already populated
- GAM init at build time: fresh creates Repository, admin user, and base data; existing detects them and skips creation

---

## Recipe Steps
The Nexa-mechanic steps (KB creation, file writing/import, build, create DB) are delegated to Nexa. This list shows the GAM-specific actions

- Query the existing GAM DB for Repository GUID, Application GUID, and connection user (see the SQL in "The Database is the Source of Truth" above)
- Decide credentials privacy with the user (see "Step 0: Ask About Credential Privacy" above)
- GAM properties to write in `<KBName>.kb.gx` `#Version`:
	* `EnableIntegratedSecurity = "True"`
	* `IntegratedSecurityLevel = "Authentication"` or `"Authorization"`
	* `ApplicationId = "<existing AppGUID from gam.Application>"` ← the key change versus fresh, where you OMIT it
	* `LoginObjectForWeb = "GAMExampleLogin"` or a custom WebPanel
- GAM properties to write in `<EnvName>.local.env.gx` `#Properties`:
	* `RepositoryId = "<existing RepGUID from gam.Repository>"` ← THE KEY PROPERTY
	* `AdministratorUserName = "<admin name from gam.User>"`
	* `AdministratorUserPassword = "<admin password; user provides, mandatory>"`
	* `ConnectionUserName = "<existing RepConUser, case-sensitive>"`
	* `ConnectionUserPassword = "<connection user password; user provides, mandatory>"`
- GAM DataStore connection in `<EnvName>.local.env.gx` `#Backend`: point the `GAM` DataStore at the EXISTING GAM database name and server
- Import the Version + Environment changes (names format → Nexa)
- Verify the Environment property persisted (GAM-specific check on the KB metadata DB):

	```sql
	SELECT CAST(EVP.EntityVersionProperties AS VARCHAR(MAX))
	FROM EntityVersion EV
	JOIN EntityVersionProperties EVP ON EV.EntityVersionId = EVP.EntityVersionId
	WHERE EV.EnvironmentName = '<EnvName>';
	```

	Expected: XML contains `IntegratedSecurityRepositoryId` with the GUID set in "GAM properties to write at Environment scope" above

- Build (Nexa scope). After build, verify GAM-specific outputs:
	* `<TargetPath>/Library/GAM/InputParms.json` → `"IdeConnection":{"Repository":"<existing-GUID>", …}`
	* `<TargetPath>/Library/GAM/OutputParms.json` → `{"isOK":true, "Errors":[]}`
	* `<TargetPath>/web/connection.gam` → contains the existing GAM Repository's `<Key>` GUID
- Create database: ONLY the Default DataStore DB (Nexa scope). The GAM DB already exists; GAM init detects it and skips table creation
- Smoke test: login with the existing admin credentials at `gamexamplelogin.aspx`

### What NOT to do
- Do NOT skip `RepositoryId` at Environment scope; without it, the build generates a random GUID and fails with GAM2
- Do NOT set `RepositoryId` at Version scope; it is ignored silently
- Do NOT skip `ApplicationId` at Version scope when joining an existing application; without it, the auto-generated value will not match the existing `gam.Application` row
- Do NOT write any GAM property as a quoted display name with spaces: ignored silently, then the build fails in GAM init
- Do NOT run a GAM initialization that creates a new admin user: the existing one is canonical
- Do NOT overwrite security policies, roles, or users that already exist in the GAM DB

---

## Fallback: Post-Build `connection.gam` Replacement
Use ONLY when `RepositoryId` cannot be set at import time; e.g. a prior build already generated a wrong `connection.gam` and the user wants a quick patch without re-importing. The clean path is always "Recipe Steps" above

### Get the connection key
- Option A, query the existing GAM DB: `SELECT RepConKey FROM gam.RepositoryConnection WHERE RepConName = '<connection-name>'`
- Option B: export from GAM Backoffice: Repository → Connections → FILE button → download `connection.gam`
- Option C: user provides the key directly

### Patch the file
Overwrite `<TargetPath>/web/connection.gam`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Connection>
	<key>&lt;existing-connection-key-guid&gt;</key>
</Connection>
```

CRITICAL: this patch is overwritten on the NEXT build. For a permanent fix, set `RepositoryId` per "Recipe Steps" above and rebuild

---

## Validated Path Summary
The 5 GAM Environment-scope properties + `ApplicationId` at Version scope + targeted import + build produces:

- `OutputParms.isOK:true`
- `InputParms.Repository` matches the existing `RepGUID`
- `connection.gam` correct on first build
- No IDE UI required, no post-build patching of `connection.gam`
