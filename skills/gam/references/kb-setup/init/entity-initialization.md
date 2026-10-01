---
name: gam-entity-initialization
description: Master skill for declarative GAM entity initialization. Covers the 3-Object Trio pattern (SDT + DataProvider + Procedure), idempotency strategies, logging, dependency order, orchestrator, and API quirks. Links to per-entity code-providers for actual code
---

# GAM Entity Initialization — Declarative Pattern
Validated against production GoodPractices XML. This is the only acceptable
pattern for initializing any GAM entity in a repeatable, environment-independent way

---

## MANDATORY First Step — Read the External Object
Applies the [EO Discovery Protocol](../../global-constraints.md#eo-discovery-protocol-mandatory) to GAM entities specifically: code-providers in `code-provider/` describe STRUCTURE and idempotency strategy, NOT the property source — the EO file in the KB always wins

### Where to find EO definitions
**In GeneXus IDE:**
Objects panel → module `GeneXusSecurity` → External Objects → open target EO
(e.g., `GAMUser`, `GAMApplication`, `GAMRole`)

**In KB text files (via `gx` or filesystem):**
See [global-constraints.md § Where GAM's EOs live](../../global-constraints.md#where-gams-eos-live): files resolve under `<kb-src-dir>/ref/GeneXusSecurity/`, one per object, **flat or nested in submodules depending on the module version**. Resolve the layout from `ref/GeneXusSecurity/module.toml` first, then search the tree RECURSIVELY. **Do not assume a fixed file-name suffix** (Nexa owns and may change this export convention); match the file whose content header declares `ExternalObject <Name>`
For the catalog of GAM EO names, roles and the legacy → modular mapping, see [global-constraints.md § GAM EO catalog](../../global-constraints.md#gam-eo-catalog--names-and-roles-only-never-signatures)
Qualified-name syntax for `DataType` and for calls: [global-constraints.md § Qualified-name syntax](../../global-constraints.md#qualified-name-syntax-modular-layout)

### How to read an EO file
Each file has two sections:

**`#ExternalProperties`** — all settable/readable properties:
```
#ExternalProperties
	GUID [
		PropertyType = 'Read/Write',
		BasedOn = 'GAMGUID, GeneXusSecurityCommon',
		...
	]
	IsBlocked [
		PropertyType = 'Read/Write',
		BasedOn = 'GAMBoolean, GeneXusSecurityCommon',
		...
	]
	NameSpace [
		PropertyType = 'Read Only',
		...
	]
```

Key fields to read:
- `PropertyType`: `'Read/Write'` (settable) or `'Read Only'` (computed)
- `BasedOn`: the GeneXus domain type — use this in SDT-Def `DataType` fields

**`#ExternalMethods`** — all callable methods:
```
#ExternalMethods
	GetTypeByName [
		IsStatic = 'True',
		NetFrameworkExternalType = 'String',   // return type
		...
	]
	{
		Parameters
		{
			Name   [ AccessType = 'In',  BasedOn = 'GAMDescriptionShort, ...' ]
			Errors [ AccessType = 'Out', Type = 'GAMError, GeneXusSecurity'  ]
		}
	}
```

Key fields to read:
- `IsStatic = 'True'` → called as `GAMUser.GetByGUID(...)` (class-level)
- `IsStatic = 'False'` / absent → called on instance `&GAMUser.Save()`
- `NetFrameworkExternalType` / `JavaExternalType` → return type (`void`, `bool`, `String`, etc.)
- `Parameters.AccessType = 'In'` → input parameter; `'Out'` → output (errors collection)

> **When in doubt about a property name or method signature, read the EO file itself
> (located per "Where to find EO definitions" above) — it is the single authoritative source.**

### SDT Generation Workflow — from EO to SDT
The canonical workflow lives in
[global-constraints.md → SDT Generation Workflow](../../global-constraints.md#sdt-generation-workflow--derive-the-sdt-11-from-the-eo)
6 steps, in order, every time an SDT is needed. Hard rule:

> Property count in the SDT = count of `Read/Write` properties in the EO
> (plus any `Read Only` properties explicitly required for the use case)

### Nested entities — generate SDTs on demand
No dedicated code-providers exist for the entities listed below. When initializing
a parent (e.g., `GAMApplication`) requires creating instances of these, derive each
nested SDT by applying the SDT Generation Workflow against the corresponding EO

- `GAMApplicationMenu` (`GAMApplication.GetMenus`, `AddMenu`)
- `GAMApplicationMenuOption`
- `GAMApplicationPermission` (`GAMApplication.GetPermissions`, `AddPermission`)
- `GAMPermission` (role assignment — different shape than `GAMApplicationPermission`)
- `GAMPermissionFilter` (input filter for `GetPermissions`)
- `GAMProperty` (key/value used by roles, users, applications)
- `GAMAuthenticationType` and subtypes (`GAMAuthenticationTypeOAuth20`, `GAMAuthenticationTypeSAML20`, …)

All live under `ref/GeneXusSecurity/`, and in the modular layout filter and sub-entity EOs sit in the `Common` submodule of their owner. Locate each by content header (see "Where to find EO definitions" above), not by an assumed file name or folder

### Looking up sub-object properties
Some EO properties return sub-objects (e.g., `GAMApplication.Environment`). To find
their properties:

- In the `GAMApplication` EO, find the `Environment` property entry
- Read its `BasedOn` or `NetFrameworkExternalType` field — this gives the sub-type name
- Open that sub-type's EO file (same folder, same lookup rule) for its `#ExternalProperties`

### Methods not listed in individual EO files
Some methods are inherited from the base EO class and do NOT appear in the individual
EO files:

- `.ToJsonString()` — available on ALL GAM EOs; serializes the full object to JSON; useful for diagnostic error logging
- `.Success()` / `.Fail()` / `.GetErrors()` — base class methods present on all EOs that extend `ExternalObjectBC`

> `GAMAuthenticationType` does NOT extend `ExternalObjectBC` — it has no `Load/Save/Success`

---

## The 3-Object Trio
Every entity initialization requires exactly three GeneXus objects:

- `<Entity>Definition` — SDT: declares the shape of the init data
- `Init<Entity>Data` — DataProvider: provides values (swap per environment)
- `Init<Entity>GAM` — Procedure: applies values to GAM idempotently

One trio per entity. The orchestrator (`InitGAMEntities`) calls them in dependency order

> **Mandatory — always use the SDT + DataProvider + Procedure trio.**
> Default (consolidated): per-entity SDTs + ONE Master SDT (`GAMInitializationDefinition`) + ONE master DataProvider + ONE master Procedure. Every top-level entity in the Master SDT is `Collection = 'True'` — even when initializing a single instance. See [GAM Entity Initialization — Consolidated Pattern](entity-initialization-consolidated.md)
> Alternative (individual): one SDT + one DataProvider + one Procedure per entity
> Never write a single Procedure with inline variable assignments — the DataProvider is the
> environment-swap point and the single place to change values per deployment

---

## GX Text Format Rules (Critical)
```
SDTs: do NOT author a #Properties section (see note below)
SDTs: define sub-types as SEPARATE SDT objects; reference by DataType name
Collection members: Collection = 'True' (string, not boolean literal)
Procedures: Parm() declarations go in #Rules, NOT in #Properties
Every object: IntegratedSecurityLevel = "SecurityNone" in #Properties
Non-translatable strings: always prefix with !"  e.g. !"local", !"admin"
DataProvider: Output = "<MasterSDTName>" (consolidated) or "<EntitySDTName>" (individual) in #Properties; no Parm declarations
Sub routines: Do "Name" / Sub "Name" ... EndSub (no Parm inside Sub)
```

> **On the SDT `#Properties` rule:** the constraint is on AUTHORING, not on the block's legitimacy. Do not write it, because an authored `#Properties` block is the reported cause of a `NullReferenceException` on validate. GeneXus generates it itself on re-export (`Name` and `IsDefault`), so finding it in a round-tripped SDT is expected and must not be removed or reported as an error

---

## GAM Domain Types
All GAM domains live flat in `GeneXusSecurityCommon`, exported to `ref/GeneXusSecurityCommon/#domains/`. Read the Domain file before comparing, assigning or sizing a value, see [global-constraints.md § Domain Discovery](../../global-constraints.md#domain-discovery-mandatory-alongside-eo-discovery). Types below are verified; anything not listed must be read, not assumed

- `GAMGUID`: `Character(40)`, all GUID fields
- `GAMDescriptionShort`: `Character(60)`
- `GAMDescriptionMedium` and `GAMDescriptionMediumVC`: medium descriptions
- `GAMDescriptionLong`: `Character(254)`
- `GAMKeyNumLong`: long integer IDs
- `GAMKeyNumShort`: short integer IDs, returned by `GAMRepository.GetId()`
- `GAMBoolean`: **real `Boolean`**, not `Character(1)`; write `If &Flag`, never a comparison against `!"1"` or `!"Y"`
- `GAMEvents`: `Character(60)` enumerated, the authoritative event catalog
- `GAMEventSubscriptionStatus`: `Character(1)` enumerated, `Unsubscribed` (`u`) or `Subscribed` (`s`)
- `GAMClientApplicationId` and `GAMClientApplicationSecret`: application client_id and client_secret
- `GAMPermissionAccessType`: permission type enum
- `GAMMenuOptionType`: menu option type enum
- `Url` (GeneXus): URLs and GeneXus object names

For an enumerated Domain, `EnumValues` holds triples of name, translation key and STORED value; use dot notation on the name (`GeneXusSecurityCommon.GAMPermissionAccessType.Allow`) and never write the stored value as a literal

Use `VarChar(n)` / `Boolean` / `Integer` if a domain is not resolvable in the target KB

---

## MyGUIDs — Stable GUID Constants (OPTIONAL)
> **OPTIONAL.** GAM auto-generates a GUID for every entity on `Save()`. Define `MyGUIDs`
> only when you need stable, predictable GUIDs across environments (dev / QA / prod) for
> cross-environment idempotency lookups. Omit this domain entirely for single-environment
> deployments or when name / email-based idempotency is sufficient

Define a `MyGUIDs` enumerated `Domain` (type `VarChar(40)`) with all application
GUIDs as named constants. This eliminates magic strings and ensures consistency
across dev / QA / production

```genexus
Domain MyGUIDs
{
	MyApp = !"a1b2c3d4-e5f6-7890-abcd-ef1234567890"
	RoleSystemAdministrator = !"role-sysadm-0000-0000-000000000001"
	RoleEmployee = !"role-empl-00000-0000-000000000002"
	RoleCustomer = !"role-cust-00000-0000-000000000003"
}
```

Usage in DataProvider:
```genexus
GUID = MyGUIDs.MyApp
```

Usage in Procedure:
```genexus
&GAMRole = GAMRole.GetByGUID(MyGUIDs.RoleSystemAdministrator, &GAMErrors)
// EnumerationDescription returns the label string
&Text = !"Role: " + MyGUIDs.EnumerationDescription(MyGUIDs.RoleSystemAdministrator)
```

---

## Idempotency Strategies
> **GUID is auto-generated by GAM on `Save()` for every entity.** Do not set `GUID` in
> the DataProvider. Use name / email-based idempotency when no stable GUID is pre-known
> Only pre-assign GUIDs when using the optional `MyGUIDs` domain (see "MyGUIDs — Stable GUID Constants (OPTIONAL)" above)

> **Identifier casing below is the flat-layout spelling** (`Id`, `SecurityPolicyId`, `MainMenuId`). The
> modular layout re-cased most of them (`ID`, `SecurityPolicyID`, `MainMenuID`) but not all; resolve the
> layout and read the EO before writing the member. See
> [global-constraints.md § Member casing changed with the layout](../../global-constraints.md#member-casing-changed-with-the-layout)

- **A — GUID** (Application, Role): `GetByGUID(guid, errors)` → if found: `Load(Id)`; else: `new()`. GUID omitted from DataProvider — GAM auto-generates on `Save()`. When no GUID: use name-based lookup (`GetRoles` / `GetApplications`)
- **B, filter lookup** (SecurityPolicy, EventSubscription): resolve the identifier through the entity's list/filter method, then `Load(<id>)` if found, `new()` if not. The identifier is assigned by GAM, never set or hardcode it. There is no static getter for it, so a NEW entity has nothing to load: the branch must start at `new()`. For EventSubscription the lookup is `GAMRepository.GetEventSubscriptions(&EventSubscriptionFilter, &GAMErrors)` matched on `Event` + `ClassName`, see [GAMEventSubscription Code Provider](code-provider/gameventsubscription-code-provider.md)
- **C, Singleton** (Repository): `GetId()` → `Load(id)` (static, `BasedOn = 'GAMKeyNumShort'`); never assign the id directly. `GAMRepository.Get()` is the alternative when the instance is wanted without the round trip
- **D — GUID (void Load)** (User): `Load(guid)` void → check `GUID.IsEmpty()` (NOT `Success()`). GUID and Login omitted from DataProvider — GAM auto-generates on `Save()`. When no GUID: use `GetUsers(filter.Email)`

> **Critical for User (Strategy D):** `GAMUser.Load(guid)` is void — do NOT assign
> to `= new()` before calling. After Load, check `&GAMUser.GUID.IsEmpty()` to
> determine if the user exists. Using `not &GAMUser.Success()` here is WRONG
> and causes GAM78 (duplicate email) on the second run

---

## Logging Pattern
Use `Log.Write` — NOT `Msg(..., status)`. `Msg` does not work in headless/batch
execution and does not integrate with GeneXus log infrastructure

`Log` is `GeneXus.Common.Log` (`ref/GeneXus/Common/Log.gx`); qualify the object, not only the level:

```genexus
&Text = !"[Init] Saving: " + &AppDef.Name
GeneXus.Common.Log.Write(&Text, !"$" + &Pgmname, GeneXus.Common.LogLevel.INFO)
```

Error log:
```genexus
GeneXus.Common.Log.Write(&Text, !"$" + &Pgmname, GeneXus.Common.LogLevel.ERROR)
```

The overload used above is `Write(message, topic, logLevel)`. `Log.gx` declares four `Write`
overloads plus `Info` / `Warning` / `Error` / `Debug` / `Fatal` shortcuts; read the EO before using a
different one

`&Pgmname` and `&Pgmdesc` are standard implicit variables: GeneXus omits them from the re-exported
`#Variables` block. Their absence after a round trip is not a lost declaration, so do not re-add them

---

## Initialization Order (Dependency Graph)
```
1. GAMRepository        ← singleton, no dependencies
2. GAMSecurityPolicy    ← no dependencies
3. GAMApplication       ← creates Permissions and Menus; Role.AddPermission() needs these permission GUIDs to already exist
4. GAMRole              ← references SecurityPolicyId (optional) and Application Permission GUIDs (see gamrole-code-provider.md)
5. GAMUser              ← references AuthenticationTypeName; role assignment requires Roles to exist
6. GAMEventSubscription ← calls GAMRepository.SubscribeEvent after Save
7. ClearLastErrors() + ClearCache()   ← ALWAYS last
```

Step 7 in code: `ClearLastErrors` returns `GAMBoolean` and needs an assignment target;
`ClearCache` is `System.Void`:

```genexus
&Cleared = GAMRepository.ClearLastErrors()
GAMRepository.ClearCache()
```

---

## Scope — Which Entities to Initialize
When a user asks to initialize GAM, confirm scope before generating code

- `full` (DEFAULT) — Repository + SecurityPolicy + Roles + Users + Application + EventSubscriptions
- `users_only` — only create / update users and assign existing roles
- `roles_only` — only create roles; skip users and applications
- `auth_only` — only configure authentication types on an existing repo
- `custom` — user specifies exactly which entities

When scope is `custom`, load only the code-providers for the requested entities (see "Code Providers (per entity)" below)
When scope is `full` (default), follow the initialization order from "Initialization Order (Dependency Graph)" above

---

## Initialization Pattern — Consolidated vs Individual
### DP-Init1: Consolidated or Individual?
- Trigger: Before generating any object. Always
- Question: "Pattern — individual trio per entity, or one consolidated DataProvider + Procedure for all entities?"
- Options:
	* `consolidated` (DEFAULT) — per-entity SDTs (with child SDTs for nested entities) + ONE master DataProvider + ONE master Procedure for all entities. Best for production, full-KB init, CI/CD deploys, and single-entity requests. Switch to [GAM Entity Initialization — Consolidated Pattern](entity-initialization-consolidated.md)
	* `individual` — one SDT + DataProvider + Procedure per entity. Best for learning, incremental setup, or when explicit per-entity isolation is preferred. Continue with the rest of this file
- Impact: Selects the object-count footprint and orchestration model. Both patterns share idempotency strategies, the EO Discovery Protocol, and dependency order
- Phase: design

---

## Code Providers (per entity)
Load the relevant code-provider when working on a specific entity
Each file contains the SDT(s), DataProvider, and Procedure for that entity

- `GAMRepository` — C, Singleton: [gamrepository-code-provider](code-provider/gamrepository-code-provider.md)
- `GAMSecurityPolicy`: B, filter lookup, [gamsecuritypolicy-code-provider](code-provider/gamsecuritypolicy-code-provider.md)
- `GAMRole` — A, GUID: [gamrole-code-provider](code-provider/gamrole-code-provider.md)
- `GAMUser` — D, void Load: [gamuser-code-provider](code-provider/gamuser-code-provider.md)
- `GAMApplication` — A, GUID: [gamapplication-code-provider](code-provider/gamapplication-code-provider.md)
- `GAMEventSubscription`: B, filter lookup, [gameventsubscription-code-provider](code-provider/gameventsubscription-code-provider.md); handler contract: [domain-event-subscription](../../events/domain-event-subscription.md)

---

## Runtime Detection Utilities
Use these when the init procedure needs to behave differently based on runtime context

### Detect Web vs Command Line
```genexus
// Procedure: IsWebApplication — Parm(out: &IsWebApplication)
&IsWebApplication = False
java   if (com.genexus.ApplicationContext.getInstance().isServletEngine()) {
csharp if (GxContext.IsHttpContext) {
	&IsWebApplication = True
java   }
csharp }
```

### Auto-Detect Application Base URL
```genexus
&CurrentURL = GAMHelper.GetCurrentApplicationBaseURL()
```

### Detect Generator at Runtime
```genexus
&GAMGeneXusGenerator = GAMHelper.GetGeneratorLanguage()
If &GAMGeneXusGenerator = GeneXusSecurityCommon.GAMGeneXusGenerator.Java
	// Java-specific path
Else
	If &GAMGeneXusGenerator = GeneXusSecurityCommon.GAMGeneXusGenerator.NetFramework
		// .NET Framework path
	Else
		// .NET Core path
	EndIf
EndIf
```

---

## Orchestrator: InitGAMEntities (Consolidated pattern)
For the consolidated pattern (DEFAULT), the orchestrator receives the Master SDT from `InitGAMData()` and, in the dependency order from "Initialization Order (Dependency Graph)" above, iterates each entity collection (`For &<Entity>Definition in &Master.<Entities>`) calling `Do "Init<Entity>"` per instance, except Repository, which is a scalar member with no loop. `ClearLastErrors()` (assigned, since it returns `GAMBoolean`) then `ClearCache()` run once, always last

For the individual pattern, each entity has its own trio (`InitXxxGAM()`, `InitXxxData`, `XxxDefinition`); the orchestrator calls each procedure in dependency order

---

## API Quirks and Gotchas
- `SetClientSecret` — must be called AFTER `Application.Save()` — it is a method, not a property
- `GAMUser.Load(guid)` — void — do NOT `= new()` before. Check `GUID.IsEmpty()` after (NOT `Success()`)
- `GAMRole.Properties.Clear()` — call before re-setting Properties, or old values persist across runs
- `DeleteAllPermissions()` before `AddPermission()` — ensures idempotency — without delete, re-run creates duplicates
- `DeleteAllMenus()` before `AddMenu()` — same pattern as permissions
- `GAMPermission` ≠ `GAMApplicationPermission` — `GAMApplicationPermission` = app-level definition; `GAMPermission` = role assignment. Different EOs with different fields
- `AccessType.FromString(...)` — `GAMApplicationPermission.AccessType` is set via `.FromString()`, not direct assignment
- `isTRN` permission naming — name MUST be `<prefix>_<suffix>` — procedure splits on `_` to derive 4 child names
- `ClearLastErrors()` before `ClearCache()`: both must be called; `ClearLastErrors` comes first. `ClearLastErrors` RETURNS a value (`GAMBoolean`), so assign it or the line does not compile; `ClearCache` is void
- `GAMApplication.MainMenuId` — update in a second `Load + Save` AFTER all menus are created
- `SubscribeEvent`: call after `EventSubscription.Save()` + Commit; returns `GAMBoolean`, so assign and check the result. `Save()` alone leaves the subscription inactive, see [GAMEventSubscription Code Provider](code-provider/gameventsubscription-code-provider.md)
- `GAMErrors.Clear()` — call after handling errors in a loop to avoid stale errors affecting next operation
- No `Rollback` in `Else` of `Save()` — GeneXus performs automatic rollback when `Save()` fails. An explicit `Rollback` inside the `Else` block is redundant — the transaction is already rolled back at that point. Correct pattern: `Else / &GAMErrors = &Entity.GetErrors() / Do "LogErrors"` — no `Rollback`
