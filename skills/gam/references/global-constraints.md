---
name: global-constraints
description: Non-negotiable rules, architectural patterns, and protocols for GAM diagnostics and configuration
---

# GAM Global Constraints
Rules and patterns that apply across all GAM domains. Load this reference when creating or modifying GAM artifacts, or when performing diagnostics

---

## Architectural Patterns
### Silent Skip Anti-Pattern
When the Token Response has an empty field (e.g., `token_type`), the GAM guard `If not &field.IsEmpty()` prevents adding the header. No error and no trace are produced

Detection: Absence of `GAMRemote AddHeader` after a successful Token Response

Fix: Ensure the IDP always returns `token_type: Bearer` in the token response

### Timeout Conflict
CRITICAL: `OauthTokenExpire` (DB) MUST BE >= Web Server Session Timeout (IIS/Tomcat). If not, the UI appears logged in but AJAX calls fail with session expired

Detection: Error Code 17 or 114 appearing intermittently after successful login

Fix: Align both values — either increase `OauthTokenExpire` or decrease the web server timeout

### Explicit Deny Override
In permission evaluation, an explicit Deny in ANY role ALWAYS overrides a Grant in another role. Implicit Deny (not assigned) is different from Explicit Deny

### State Snapshot Mechanism (SLO Chaining)
GAM persists context across redirects via the `state` parameter stored in the database. On callback, it loads the snapshot by hash of the state parameter

### Trace-to-Behavior Correlation
Traces are emitted at specific decision points inside GAM. When a trace is absent, it means the corresponding condition was not met. Observable parameter values in trace output indicate which flow phase was executed

Key principle: The ABSENCE of a trace is stronger evidence than an error message

---

## Deployment Constraints
### IIS Virtual Directory (DEFAULT — always)
Every KB deploys as an IIS virtual directory, NEVER standalone on localhost:port. Name: `<KBName><EnvironmentSuffix>`. If the user does not specify a deployment method, ALWAYS use virtual directory

### IIS AppPool SQL Permissions (MANDATORY)
When deploying via IIS with Trusted Connection, the identity `IIS AppPool\DefaultAppPool` MUST have a login in SQL Server + `db_owner` on BOTH databases (Default and GAM). Without this: HTTP 500 with no visible error in the browser — only in stdout logs. This is the #1 most common error when deploying KBs

### DB Name Mismatch (GX_KB_ prefix)
GeneXus may create the database as `GX_KB_<KBName>` but the config says `<KBName>`. Verify post-build that the names match

---

## Diagnostic Protocol
### Trace Activation (3-step)
Traces require ALL THREE steps to be active — both the System Parameter AND the Repository Property are boolean `0`/`1` only (never `2`/`3`). Full procedure, file locations, and cache-expiry caveat: [Activating GAM Traces (3-Step Process)](./debugging/common-debugging.md#activating-gam-traces-3-step-process)

If ANY step is missing, traces will be silent

### Diagnostic Output Format / Ask Before Acting
Canonical in `SKILL.md` § OUTPUT (action-first default shape, extended Root Cause → Evidence → Code Path → Recommendation → Confidence format on request, ask-before-acting rules). Do not restate here

### Cross-Generator Awareness
If an issue could differ between .NET Framework / .NET Core / Java, always mention this and reference [GAM Cross-Generator Issues Encyclopedia](./environments/domain-cross-generator.md)

---

## Code Conventions
- Use `!"…"` for non-translatable strings in GeneXus code
- Use dot notation for enum values (e.g., `GAMRepository.Default`)
- No `ElseIf` — use separate `If` blocks or `Do Case`
- For GeneXus code fundamentals, consult the Nexa skill
- When a fix requires writing GeneXus code, delegate to the Nexa skill
- **GAM entity initialization**: ALWAYS use the 3-object trio — SDT (`<Entity>Definition`) + DataProvider (`Init<Entity>Data`) + Procedure (`Init<Entity>GAM`). Never write a single Procedure with inline variable assignments. See [GAM Entity Initialization — Declarative Pattern](./kb-setup/init/entity-initialization.md)
- **No explicit `Rollback` in `Else` of `Save()`**: GeneXus performs automatic rollback when `Save()` fails. An explicit `Rollback` inside the `Else` of `If &Entity.Success()` is redundant and incorrect. Correct: `Else / &GAMErrors = &Entity.GetErrors() / Do "LogErrors" / EndIf` — no `Rollback`

---

## User-Facing Communication Rules
### No Internal API Exposure
Never include in responses to the user:
- Internal GAM variable names (e.g., `&Parm_isServerIP`, `&TokenToFinish`, `&haveChildren`)
- Internal procedure names as implementation references (e.g., "in procedure GAMExternalAuthenticatinOAuth20, line 78222")
- Pseudocode or For Each queries against internal GAM database tables
- Source code line numbers or XML file references
- Internal state machine logic or conditional branching code

Exception: Trace marker strings that appear in log output (e.g., `GAMTrace-SLOProcess - Parm_isServerIP:1`) are observable by the user and MAY be referenced — but without showing the source code that emits them

### User's Actionable Surface
All recommendations must be achievable using ONLY:
- Backoffice — GAM Web Administration UI
- External Objects — GAM EOs and their public methods (readonly surface)
- Public API methods — Methods documented and exposed to the developer
- Configuration files — connection.gam, log.config, web.config, appsettings.json
- Public endpoints — /oauth/gam/signin, /oauth/gam/access_token, etc
- Traces/Logs — Read-only diagnostic output
- Diagnostic SQL — Queries against the GAM database for inspection (marked as such)

If a root cause is an internal GAM bug, state the behavior observed and recommend: workaround, configuration change, or contacting GeneXus support — never suggest modifying internal GAM code

---

## Version Scope
GAM's programmable surface is versioned by the `GeneXusSecurity` module, not only by the product; resolve both before applying a scoped reference or writing EO code

- Primary axis: `GeneXusSecurity` `ModuleVersion` from `ref/GeneXusSecurity/module.toml`, which decides the EO layout (see "Where GAM's EOs live") and the availability of newer EO members
- Secondary axis: `ProductVersion` from `*.kb.gx`, format `<major>.<minor>.<patch>.<build>`
- Also read the module versions declared in the object's `#References` block when a generated artifact must target a specific one

This skill targets `ProductVersion: >=18.0`; it ships as part of GeneXus 4 Agents, which supports only v18 Knowledge Bases. Knowledge about earlier versions stays as background for the user, never as something to generate against

A reference narrower than that states its own scope in frontmatter, under `metadata` because these are non-standard fields:

```yaml
metadata:
  product: "ProductVersion>=18.0"
  modules:
    - "GeneXusSecurity>=3"
```

- `product` is a range over the KB `ProductVersion` from `*.kb.gx`; `modules` is a list of ranges over a module `ModuleVersion` from `ref/{module}/module.toml`
- Operators `=`, `>`, `>=`, `<`, `<=`; segments compared numerically left to right; multiple entries evaluate as logical `AND`
- A reference without these fields is cross-version within the skill target
- Reject an unsupported feature instead of inferring compatibility, and phrase the commercial product name in responses, never the raw version number

**Scope the naming, not the feature.** `GeneXusSecurity>=100` covers only the EO renaming and reorganization, which is currently in beta. GAM functionality itself is available on any `GeneXusSecurity>=3`, so never declare `>=100` on a reference whose feature ships in release versions; state the EO naming caveat in the body instead. `product` does not discriminate the layout either, since both namings ship under the same product build

---

## EO Discovery Protocol (mandatory)
GAM's entire programmable surface is exposed through its External Objects (EOs). EOs are
self-documented — every property, method, and parameter carries its own description (the GAM team
is progressively completing these descriptions; the structural signatures are always present even
where descriptions are still pending). This skill never freezes EO signatures in its references —
every property or method reference must be verified live against the KB's EO, every time

**Never write an SDT, DataProvider, or Procedure that references a GAM External Object property or
method without first reading the actual EO definition in the KB.** SDTs are the highest-risk
artifact — regenerate every SDT from the EO, never from a remembered shape. Code-providers in
`kb-setup/init/code-provider/` describe STRUCTURE and idempotency strategy — they do NOT carry the
property list

### How to read an EO — the mechanics live in Nexa
Reading and introspecting a GeneXus External Object is a Nexa capability, not a GAM one:

- EO text/source format — `#ExternalProperties` / `#ExternalMethods` / `#ExternalEvents` regions,
  parameter `AccessType`, and the `Description` field carried by every member: Nexa skill →
  `references/object-external-object.md`
- Where exported objects land — `src/` vs. the read-only `ref/` package tree, plus the companion
  `<name>.doc.md` for long-form documentation: Nexa skill → `references/global-output.md`
- The open KB / export / read workflow: Nexa skill (`gx`)

Use Nexa's mechanism to locate and read an EO — do not re-derive the reading mechanics here

### Where GAM's EOs live
GAM EOs ship in the `GeneXusSecurity` module. Once a KB has GAM built and activated, and has been exported to text, they resolve under:

	<kb_path>/ref/GeneXusSecurity/

(or under `src/` if the KB owns or extends the module). The `ref/GeneXusSecurity/` folder does not exist until export has run; its absence means EO definitions are not yet verifiable

**Search recursively.** Depending on the module version the EOs are either flat in that folder or nested in submodules; never assume a single level

**Resolve the layout first:** read `ref/GeneXusSecurity/module.toml` → `ModuleVersion`
- major `>=100` (e.g. `103.16.276`) is **modular**, currently in beta: EOs have no `GAM` prefix and live in submodules (`Tenants/`, `Tenants/Common/`, `Users/`, `Roles/`, `Sessions/`, `Applications/`, `AuthenticationTypes/`, `Manager/`, `Common/`), each level carrying its own `module.toml`
- major `<100` (e.g. `3.16.26`) is **flat**, what every release version ships: each EO is `GAM<Name>` directly under `ref/GeneXusSecurity/`

This is a naming and organization change only; it gates no GAM functionality

**Do not assume a fixed file-name suffix for the EO file** (e.g. do not hardcode `.externalobject.main.gx`); Nexa owns and may change this export convention, so check Nexa's `references/global-output.md` for the current pattern. To locate a specific EO reliably, search the tree and identify the file whose content header declares `ExternalObject <EO_name>`, never guessing the file name from the object name alone

### Qualified-name syntax (modular layout)
Read the `module.toml` of every level in the chain and join their `Name` values to build the qualifier the text format requires; the folder path is the qualifier, with `/` becoming `.`

- In `#Variables`: `DataType = 'EventSubscription, GeneXusSecurity.Tenants.Common'`, `DataType = 'Error, GeneXusSecurity.Common'`, `DataType = 'GAMEvents, GeneXusSecurityCommon'`
- In code: `GeneXusSecurity.Tenants.Repository.SubscribeEvent(…)`, `GeneXusSecurityCommon.GAMEvents.User_Insert`
- Domains keep their `GAM` prefix and stay flat in `GeneXusSecurityCommon` in both layouts

### Member casing changed with the layout
The modularization also re-cased identifier members, so the same entity has a different spelling per layout

- flat: `Id`, `ExternalId`, `SecurityPolicyId`, `MainMenuId`, `MenuId`, `AddRoleById`, `GetByExternalId`, `GetByClientId`
- modular: `ID`, `ExternalID`, `SecurityPolicyID`, `MainMenuID`, `MenuID`, `AddRoleByID`, `GetByExternalID`, `GetByClientID`

GeneXus is case-sensitive here and the wrong spelling does not compile. **The rule is not mechanical:** `User.SecurityPolicyId` and `User.DefaultRoleId` keep the lowercase form even in the modular layout. Never derive the casing from the layout alone nor from another EO; read the target EO's `#ExternalProperties` and `#ExternalMethods` every time. Prose in this skill that spells a member one way is scoped to the layout it was written against, and the EO wins

### Domain Discovery: mandatory alongside EO discovery
Enumerated values, real data types and lengths are NOT in the EOs; they are in `ref/GeneXusSecurityCommon/#domains/`, where every `BasedOn` in an EO points

Before writing code that compares, assigns or sizes a GAM value, read the Domain file:
- `DataType` and `Length` decide the declaration and the truncation risk
- `EnumValues` is the only authoritative list of allowed values; its third field is the STORED value GAM passes at runtime, the first is the name to use in dot notation
- A Domain declared `Boolean` is a real Boolean (`If &Flag`), not a `Character(1)` compared to a literal

Never hardcode an enumerated value list in generated code or in an answer; reference the Domain

### GAM EO catalog — names and roles only, never signatures
- `GAMRepository` — the tenant/repository singleton: default policy, default auth type, email/SMTP sub-object
- `GAMSecurityPolicy` — password rules, session/token timeouts, lockout policy
- `GAMApplication` — client application: OAuth/GAMRemote flags, permissions, menus, scopes
- `GAMApplicationPermission` / `GAMPermission` / `GAMPermissionFilter` — permission definition vs. role-assignment vs. filter object (three distinct EOs — easy to confuse)
- `GAMRole` — role, its permission set, custom properties
- `GAMUser` — user identity, credentials, role assignment, block/unblock
- `GAMProperty` — generic key/value custom-attribute pattern, reused by `GAMRole`, `GAMUser`, and others
- `GAMEventSubscription` — event-handler registration (paired with `GAMRepository.SubscribeEvent`)
- `GAMSession` — active session inspection/termination
- `GAMExternalAuthenticationInput` / `GAMExternalAuthenticationOutput` — entry/exit point for external auth and SLO flows
- Auth-type EOs (`GAMAuthenticationTypeOAuth20`, `GAMAuthenticationTypeSAML20`, `GAMAuthenticationTypeLocal`, `GAMAuthenticationTypeOTP`, …) — one per configured authentication type

This catalog orients discovery. It is not exhaustive and carries no signatures — always read the EO itself

Legacy → modular mapping for the entries above (applies when `ModuleVersion >=100`):
- `GAMRepository` → `GeneXusSecurity.Tenants.Repository`
- `GAMSecurityPolicy` → `GeneXusSecurity.Tenants.SecurityPolicy`
- `GAMEventSubscription` → `GeneXusSecurity.Tenants.Common.EventSubscription`
- `GAMApplication` → `GeneXusSecurity.Applications.Application`
- `GAMApplicationPermission` / `GAMPermission` / `GAMPermissionFilter` → `GeneXusSecurity.Applications.Common.ApplicationPermission` / `GeneXusSecurity.Common.Permission` / `GeneXusSecurity.Common.PermissionFilter`
- `GAMRole` → `GeneXusSecurity.Roles.Role`
- `GAMUser` → `GeneXusSecurity.Users.User`
- `GAMProperty` → `GeneXusSecurity.Common.Property`
- `GAMSession` → `GeneXusSecurity.Sessions.Session`
- `GAMError` → `GeneXusSecurity.Common.Error`
- `GAMHelper` / `GAM` → `GeneXusSecurity.Manager.Helper` / `GeneXusSecurity.Manager.GAM`
- Auth-type EOs → `GeneXusSecurity.AuthenticationTypes.*` (e.g. `GAMAuthenticationTypeOAuth20` → `OAuth20Authentication`), their property sub-objects under `GeneXusSecurity.AuthenticationTypes.Common`
- Filter and sub-entity EOs live in the `Common` submodule of their owner (e.g. `EventSubscriptionFilter`, `UserFilter`, `RoleFilter`)

The mapping is not a substitute for discovery: resolve the layout, then read the EO. Names are not always a mechanical de-prefixing; `GAMSessionLogCheckPermissionFail` becomes `GeneXusSecurity.Tenants.Common.SessionLogCheckPermissionFail`, not a `Sessions` member

### Lookup protocol — mandatory before writing any GAM EO artifact
- **Check availability** — confirm whether `export_kb_to_text` has been run. If the user has not mentioned it, ask. Or check via `gx`/Read whether `<kb_path>\ref\GeneXusSecurity\` exists
- **Resolve version and layout:** read `ref/GeneXusSecurity/module.toml` → `ModuleVersion` (see "Version Scope" and "Where GAM's EOs live"); this determines whether names are modular or flat and which scoped references apply
- **If EOs are accessible:** locate the file for the target EO (search recursively, match by content header, not by assumed file name, see "Where GAM's EOs live" above), then read it using Nexa's `#ExternalProperties` and `#ExternalMethods` format. Use ONLY the properties and methods declared, never adding ones not found in the file
- **Read the Domains:** for every `BasedOn` and every enumerated value involved, read `ref/GeneXusSecurityCommon/#domains/` (see "Domain Discovery")
- **If EOs are NOT yet accessible**: Warn the user explicitly. Offer two paths:
	* Run `export_kb_to_text` first, then continue with verified EO data
	* Proceed based on documented knowledge — state clearly this carries risk of DataType errors or non-existent property references
- **Cross-check every EO reference in generated artifacts** against the file. If a property or method is not declared in the EO, do not include it

### SDT Generation Workflow — derive the SDT 1:1 from the EO
Apply this workflow every time an SDT is needed for a GAM entity (root entity or nested child)

- Open the `<EO>` file in `<kb_path>/ref/GeneXusSecurity/` (locate by content header, not by an assumed file-name suffix — see "Where GAM's EOs live" above)
- From `#ExternalProperties`, list every property where `PropertyType = 'Read/Write'`. Skip `Read Only` properties unless the SDT specifically needs them as input shape
- For each property, create an SDT member with:
	* `Name` = property name verbatim
	* `DataType` = property's `BasedOn` value verbatim, including module qualifier
- Identify collection relationships (entity-internal children — e.g., Permissions inside Application, Options inside Menu):
	* Method returning a collection (e.g., `GetMenus`, `GetPermissions`) → child SDT, parent member `Collection = 'True'`
	* `Add<Child>` method accepting a sub-entity → confirms 1:N; generate the child SDT
	* Property typed as a sub-EO (e.g., `Environment`) → open the sub-EO's file (same lookup rule) and recurse
- **Master SDT rule** (top-level entities in `<Name>InitializationDefinition`):
	* Every entity included as a top-level member of the Master SDT is ALWAYS `Collection = 'True'` — unconditionally, regardless of how many instances are being initialized
	* Exception: `Repository` is a scalar member (singleton — platform constraint; `new()` forbidden)
	* This rule is orthogonal to the collection-relationships rule above: entity-internal children (Permissions, Menus, …) still follow the 1:N EO rule
	* See [GAM Entity Initialization — Consolidated Pattern](kb-setup/init/entity-initialization-consolidated.md)
- For each child SDT, repeat the steps above against the child EO
- If a code-provider in this skill shows a different SDT shape than the EO yields, the EO wins. Flag the discrepancy and use the EO-derived SDT

**Hard constraints:**
- Never invent, rename, or omit properties
- Never copy a `DataType` from a code-provider when the EO file is available
- Property count in the SDT MUST match the count of `Read/Write` properties in the EO (plus any `Read Only` properties explicitly required for the use case), minus the documented exception below

**Documented exception: non-assignable `Read/Write` properties**
Some `Read/Write` properties can never be set by the caller, and including them yields an SDT with unassignable members plus, for sub-entity collections, a dead child SDT. Only these three classes may be trimmed:

- **GAM-assigned identifiers:** the entity's own id (`ID` modular, `Id` flat), assigned on `Save()`
- **Audit fields:** `DateCreated`, `UserCreated`, `DateUpdated`, `UserUpdated`
- **Unused sub-entity collections:** a collection-typed property the use case does not populate, which also removes the child SDT the recursion rule would otherwise demand

Anything else stays. State the deviation and its reason in the SDT's `description` or in the entity's code-provider; an unexplained short SDT is indistinguishable from a missed property

---

## Quality Checklist
Canonical in `SKILL.md` § QUALITY CHECKLIST. Do not restate here
