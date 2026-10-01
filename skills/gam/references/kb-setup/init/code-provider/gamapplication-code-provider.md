---
name: gamapplication-code-provider
description: Capabilities and quirks for idempotent initialization of GAMApplication. Idempotency strategy A — GUID or name lookup. Includes DPs A1-A6, ClientSecret constraint, Scopes categories, and Fixed Application IDs. EO is source of truth for properties
---

# GAMApplication Code Provider
**EO**: `GAMApplication` in `ref/GeneXusSecurity/` — locate by content header, not by an assumed file-name suffix (see [global-constraints.md § Where GAM's EOs live](../../../global-constraints.md#where-gams-eos-live))
Derive the SDT using the [SDT Generation Workflow](../../../global-constraints.md#sdt-generation-workflow--derive-the-sdt-11-from-the-eo)

Child EOs (derive nested SDTs from these when the scenario requires them), all in `ref/GeneXusSecurity/`:
- Permissions: `GAMApplicationPermission`
- Menus: `GAMApplicationMenu`
- Menu options: `GAMApplicationMenuOption`
- Role assignment: `GAMPermission` (different EO from `GAMApplicationPermission`)
- Permission filter: `GAMPermissionFilter`

---

## What you can use
- **Idempotency strategy**: A — GUID. `GAMApplication.GetByGUID(guid, &errors)` (static) → if found: `Load(Id)`; else: `new()` + set GUID. When GUID is empty (autogen): `GAMApplication.GetApplications(&filter, &errors)` by name → `Load(Id)` if found; else `new()`
- **Capabilities**: lookup by GUID or by filter, load/save, client-secret rotation, permission management (add/delete-all/list/lookup-by-name/parent-child linking), and menu management (add/delete-all/list/add-option) — read the EO for exact method names and parameters
- **Auto-generation on `Save()`**: when `GUID` and `ClientId` are left empty in the DataProvider, GAM assigns them on first `Save()`
- **Read/Write properties**: derived from EO — covers name, description, version, home object, account activation object, requires-permission flag, access-unique-by-user flag, virtual directory, base-application flag, GAMRemote flags, scope flags, OAuth options, SLO settings, and more
- **`Environment` sub-object**: `&GAMApplication.Environment` — a sub-EO with `Host`, `Port`, `SecureProtocol`, `ProgramPackage`, `VirtualDirectory`. Find the `Environment` property in the `GAMApplication` EO, read its `BasedOn` type, then open that type's EO file (same lookup rule) for its properties
- **`ToJsonString()`**: inherited from base EO class — available on all GAM EOs; useful for diagnostic logging. Does NOT appear in individual EO files
- **`GAMApplicationPermission` ≠ `GAMPermission`**: `GAMApplicationPermission` = application-level permission definition; `GAMPermission` = role assignment object. Different EOs, different properties

## Known facts (not derivable from the EO)
- `GAMApplication.Load(1)` → GAMBackOffice (platform-constant ID). `Load(2)` → the KB's own application (bound to `application.gam`). `Load()` receives a `Numeric` argument. Never look up these two apps by name or GUID

## API quirks and order-of-operations
- **`ClientSecret`**: write via `SetClientSecret(secret, &errors)` ONLY — never as a property (see GAP 1 below)
- **`ClientId`**: set as a property before `Save()` only when DP-A2 = `specific`; leave empty for autogen
- **`AccessType.FromString(...)`**: `GAMApplicationPermission.AccessType` is set via `.FromString()` method, not by direct assignment
- **`isTRN` permission naming**: name MUST be `<prefix>_<suffix>` — GAM splits on `_` to derive 4 child permissions: `_Execute`, `_Delete`, `_Insert`, `_Update`. Use `AddPermissionChild(parentGUID, childGUID, &errors)` after creating each child
- **`DeleteAllPermissions()` + `DeleteAllMenus()`**: call BEFORE re-adding for idempotency — without delete, re-runs create duplicate entries
- **`GAMApplicationMenuOption.MenuId`**: must be set explicitly to `&GAMApplicationMenu.Id` before calling `AddMenuOption`, in addition to passing it as the first parameter
- **`SubMenuID` in DataProvider**: 1-based positional index — `1` = first menu added, `2` = second, etc. The procedure must map this to the actual GAM-generated menu `Id` at runtime
- **`MainMenuId`**: requires a second `Load + Save` after ALL menus are created — cannot be set in the first Save
- **DataProvider collection items**: use `Item {}` syntax, NOT the SDT type name. Applies to `Permissions`, `Roles`, `Menus`, `Options`, and `Permissions` inside `Roles`. At the Master SDT level, `Applications` itself is also `Collection = 'True'` and uses `Applications { Item { … } }` — see [GAM Entity Initialization — Consolidated Pattern](../entity-initialization-consolidated.md)
- **`GAMErrors.Clear()`**: call after each error-bearing method in a loop to avoid stale errors affecting the next iteration

---

## Decision Points Before Configuring an Application
Ask in the order listed. Each DP may gate the next

### DP-A1: New or existing application?
- Trigger: User asks to configure / initialize a GAM Application
- Question: "Is this a new application or an existing one?"
- Options:
	* `new` (DEFAULT) — ask name; propose `ClientApp` if none given. DataProvider omits `GUID`; GAM assigns on `Save()`. Idempotency: `GetApplications(filter.Name)` → `Load(Id)` if found; else `new()`
	* `existing` — ask the GUID. Idempotency: `GetByGUID(guid)` → `Load(Id)` (Strategy A)
- Impact: Selects the idempotency branch in the Procedure and whether the DataProvider seeds `GUID`
- Phase: design

### DP-A2: GUID / ClientId / ClientSecret — auto-generated or specific?
- Trigger: Always — after DP-A1
- Question: "Use auto-generated values for GUID, ClientId, and ClientSecret, or specific ones you supply?"
- Options:
	* `autogen` (DEFAULT) — omit all three from the DataProvider. GAM assigns `GUID` and `ClientId` on `Save()`. `ClientSecret` absent → procedure skips `SetClientSecret`
	* `specific` — ask explicitly which of the three to pin. Only populate confirmed fields. `GUID` and `ClientId`: set as properties before `Save()`. `ClientSecret`: set via `SetClientSecret()` AFTER `Save()` + `Commit` — never as a property (see GAP 1 below)
- Impact: When `GUID` is pinned, idempotency switches from name-lookup (`GetApplications`) to `GetByGUID`
- Phase: design

### DP-A3: GAMRemote Type — Web vs REST
- Trigger: Always — after DP-A2
- CRITICAL — apply these interpretation rules BEFORE asking:
	* User says "enable GAMRemote" (without more) → Web only
	* User says "enable GAMRemoteREST" → REST only
	* User says both → Web + REST
	* When ambiguous → ask
- Options:
	* `web` — `ClientAllowRemoteAuthentication = True`, `ClientAllowRemoteRESTAuthentication = False`. Authorization Code + redirect. Gates DP-A4 and DP-A5
	* `rest` — `ClientAllowRemoteAuthentication = False`, `ClientAllowRemoteRESTAuthentication = True`. Resource Owner Password Credentials; no redirect
	* `both` — both flags `True`. Gates DP-A4 and DP-A5
	* `none` (DEFAULT) — both flags `False`. Local auth only
- Phase: design

### DP-A4: Callback URL (only when web GAMRemote is enabled)
- Trigger: DP-A3 = `web` or `both`
- Apply in this order — do NOT ask `ClientCallbackURLisCustom`, it is derived:
	* Is the client a GAM application?
	   - YES (GAM client) → URL = `<BaseURL>/oauth/gam/callback`, `ClientCallbackURLisCustom = False`
	   - NO (non-GAM client) → ask user for their endpoint. `ClientCallbackURLisCustom = True` (automatic)
	* Set in the DataProvider's `ClientCallbackURL` field. Add this field to the SDT when needed

### DP-A5: OAuth 2.0 mode — PKCE policy
- Trigger: DP-A3 = `web` or `both`
- Question: "Which PKCE policy for OAuth Authorization Code?"
- Options:
	* `BasicPKCE` (DEFAULT, recommended) — `ClientAllowRemoteAuthenticationOAuth20Option = GAMOAuth20Options.BasicPKCE`. Accepts clients with or without PKCE. Best for SPAs
	* `PKCE` — `GAMOAuth20Options.PKCE`. Mandatory PKCE; non-PKCE clients are rejected. Most secure for modern apps
	* `Basic` — `GAMOAuth20Options.Basic`. Classic Authorization Code, no PKCE. Legacy / server-side. Not recommended for new SPAs
- Impact: Sets `ClientAllowRemoteAuthenticationOAuth20Option`. Note: `ClientAllowOAuth20PKCEMethod` (S256 / plain) is a separate property configured at the auth-type level — not set here
- Phase: design

### DP-A6: User-data scope default (silent — no question asked)
- Trigger: Always — applied automatically based on DP-A3. No question unless the user explicitly requests additional scopes
- Options (applied silently in the Procedure):
	* DP-A3 = `web` → `ClientAllowGetUserData = True` (scope `gam_user_data`)
	* DP-A3 = `rest` → `ClientAllowGetUserDataREST = True` (scope `gam_user_data`)
	* DP-A3 = `both` → both
	* User asks for roles / additional attributes / session data / granular scopes → enable the matching Category-A boolean(s) or use Category-B `ClientAllowAdditionalScope[REST]` (see GAP 2 below)
- Impact: Without `gam_user_data`, OAuth clients cannot read user profile via standard scope
- Phase: design

---

## GAP 1 — ClientSecret: NEVER assign as a property
`ClientSecret` is `Read/Write` in the EO but writing via property does NOT apply hashing/encryption. Only `SetClientSecret(secret, errors)` correctly stores the secret

Call AFTER `Save()` + `Commit`, only when `secret` is not empty:

```genexus
&GAMApplication.Save()
If &GAMApplication.Success()
	Commit
	If not &AppDef.ClientSecret.IsEmpty()
		If &GAMApplication.SetClientSecret(&AppDef.ClientSecret, &GAMErrors)
			Commit
		EndIf
	EndIf
EndIf
```

## GAP 2 — Scopes: two distinct categories
- Category A — one `Boolean` per scope (`BasedOn = 'GAMBoolean, GeneXusSecurityCommon'`):
	* Web: `ClientAllowGetUserData`, `ClientAllowGetUserAdditionalData`, `ClientAllowGetUserRoles`, `ClientAllowGetSessionInitialProperties`, `ClientAllowGetSessionApplicationData`
	* REST mirrors: `ClientAllowGetUserDataREST`, `ClientAllowGetUserAdditionalDataREST`, `ClientAllowGetUserRolesREST`, etc
	* DP-A6 applies `ClientAllowGetUserData[REST] = True` silently based on DP-A3
- Category B — single `String`, values joined with `+` (`BasedOn = 'GAMPropertyValue, GeneXusSecurityCommon'`):
	* `ClientAllowAdditionalScope` (Web) / `ClientAllowAdditionalScopeREST` (REST)
	* For: `user_email`, `user_phone`, `user_guid`, `user_username`, `user_external_id`, custom `user_<AttributeID>`
	* Example: `&GAMApplication.ClientAllowAdditionalScope = !"user_email+user_guid"`
- DO NOT put Category A scope names inside the Category B string

---

## Cross-references
- EO: `GAMApplication` in `ref/GeneXusSecurity/`
- [SDT Generation Workflow](../../../global-constraints.md#sdt-generation-workflow--derive-the-sdt-11-from-the-eo)
- [GAM Entity Initialization — Declarative Pattern](../entity-initialization.md)
- [GAM Entity Initialization — Consolidated Pattern](../entity-initialization-consolidated.md) (default pattern)
- [GAMRole Code Provider](gamrole-code-provider.md) — Permission assignment to roles
- [Backoffice: applications](../../../backoffice/applications.md)
- [Authorization Code Flow (Web IDP)](../../../authentication/local-idp/authorization-code-flow.md) — PKCE options
