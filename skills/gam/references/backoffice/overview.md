---
name: backoffice-overview
description: GAM Backoffice structure, menu, components, code-to-backoffice flow, access, and decision points
---

# GAM Backoffice — Overview
Load this first when navigating the Backoffice or mapping UI areas to API. For field-level details of a specific section, load the matching file (`users.md`, `roles.md`, `applications.md`, `repository.md`, `security-policies.md`, `auth-types.md`, `sessions-and-events.md`)

See [GAM Entity Initialization — Consolidated Pattern](../kb-setup/init/entity-initialization-consolidated.md) for creating entities by code
See [GAM Authentication Configuration Reference](../authentication/external-providers/common/domain-auth-config.md) for auth types

---

## GAM Backoffice Structure
The GAM Backoffice is a GeneXus web application (WebPanels) organized in sections. It is accessed as a GAM Application with GUID `GAMInternalGUIDs.AppGAMUserBackend`

The public WebPanel objects ship with the `GAMExampleWW` prefix (for example `GAMExampleWWUsers`, `GAMExampleWWRoles`). The `GAM_*` identifiers used as section headings in this file are module / menu labels exposed by `GAM_GetBackendMenu`, not WebPanel object names

### Main Menu (GAM_GetBackendMenu)
```
GAM Backoffice
├── Dashboard (GAMExampleDashboard)
├── Users (GAM_Users)
│   ├── List Users
│   ├── Add/Edit User
│   └── User Detail (roles, attributes, sessions)
├── Roles (GAM_Roles)
│   ├── List Roles
│   ├── Add/Edit Role
│   └── Role Permissions
├── Applications (GAM_Applications)
│   ├── List Applications
│   ├── Add/Edit Application
│   └── Application Detail (OAuth, SSO, SLO, APIKey, MiniApp, STS, Languages, Permissions)
├── Repository (GAM_Repository)
│   ├── Repository Settings
│   ├── Change Repository (only if multi-tenant)
│   └── Repository List (only if GAM Administrator)
├── Settings (GAM_Security_Policies / GAM_Authentication_Types / GAM_General)
│   ├── Security Policies
│   ├── Authentication Types (only if not a slave repo)
│   └── Repositories (only if GAM Administrator)
├── Connections (GAM_Connections)
├── Passwords (GAM_Passwords)
├── Sessions (GAM_Sessions)
└── Event Subscriptions (GAM_Event_Susbcriptions)
```

### Menu Visibility Logic
The menu dynamically hides items based on context:
- Sessions: Hidden if the repository is a slave (AuthenticationMasterRepository)
- Change Repository: Hidden if NOT multi-tenant
- Authentication Types: Hidden if the repository is a slave
- Repositories: Hidden if the user is NOT a GAM Administrator

---

## Creating the Backoffice App by Code
Static method `GAMRepository.CreateGAMBackofficeApplication(&errors)` creates the internal backoffice application — `Commit` on success

For Java, additionally set `Environment.ProgramPackage` on the app resolved via `GAMApplication.GetByGUID(GAMInternalGUIDs.AppGAMUserBackend, &errors)` (idempotent Load-or-New shape — see [Idempotent Save Pattern](../debugging/common-genexus-patterns.md#idempotent-save-pattern-load-or-new))

---

## Common Backoffice Components
### Reusable WebComponents
- `GAM_HeaderWW` — Header with logo and user info
- `GAM_PagingWW` — Grid pagination
- `GAM_DataCard` — Reusable data card
- `GAM_GetBackendMenu` — Side menu Data Provider
- `GAM_ConvertErrorsToMessages` — Converts GAMError[] to Messages SDT
- `GAM_CheckUserActivationMethod` — Verifies activation method and sends emails
- `GAM_DisplayLastGAMErrors` — Displays latest GAM errors

### Error Handling Pattern in the Backoffice
Same idempotent Save/error shape as [Idempotent Save Pattern](../debugging/common-genexus-patterns.md#idempotent-save-pattern-load-or-new), with one Backoffice-specific step: pass `&GAMErrorCollection` through `GAM_ConvertErrorsToMessages(&GAMErrorCollection, &Messages)` before displaying `&Messages` on screen

---

## Flow: Code to Backoffice
### "I did this by code — where do I see it in the backoffice?"
- `GAMRepository.Save()` — Repository > Repository Settings
- `GAMUser.Save()` — Users > List Users
- `GAMRole.Save()` — Roles > List Roles
- `GAMApplication.Save()` — Applications > List Applications
- `GAMSecurityPolicy.Save()` — Settings > Security Policies
- `GAMAuthenticationType*.Save()` — Settings > Authentication Types
- `GAMEventSubscription.Save()` — Event Subscriptions
- `GAMApplication.AddPermission()` — Applications > [App] > Permissions tab
- `GAMUser.SetMainRoleById()` — Users > [User] > Roles tab
- `GAMUser.GenerateApplicationAPIkey()` — Users > [User] > Keys tab

### "I want to configure this in the backoffice — what is the equivalent code?"
- Users > Add User — `&GAMUser = new(); …Save()`
- Users > Edit > Set Role — `&GAMUser.SetMainRoleById(&id, &err)`
- Users > Edit > Block — `&GAMUser.IsBlocked = True; .Save()`
- Roles > Add Role — `&GAMRole = new(); …Save()`
- Roles > Add Permission to Role — `&GAMRole.AddPermission(&perm, &err)`
- Applications > Add — `&GAMApp = new(); …Save()`
- Applications > Edit > OAuth tab — Set `Client*` properties
- Settings > Auth Types > Add OAuth 2.0 — `&GAMAuthTypeOAuth20 = new(); …Save()`
- Settings > Security Policy > Edit — `&GAMSecPolicy.Load(&id); …Save()`
- Repository > Edit — `&GAMRepo.Load(&id); …Save()`

---

## Backoffice Access
### Access URL
- .NET Framework: `http://host/VirtualDir/gamhome.aspx`
- .NET Core: `http://host/VirtualDir/gamhome`
- Java: `http://host:port/context/com.myapp.gamhome`

### Requirements
- The backoffice application must exist (`CreateGAMBackofficeApplication`)
- The user must have the GAM Administrator role (role ID 1 or external_id `is_gam_administrator`)
- The user must be able to log in (auth type configured)

---

## Decision Points
Before guiding users through Backoffice operations, check which Decision Points apply

### DP-1: Intention — Read or Write?
- Trigger: User asks about the Backoffice or "where is X configured"
- Question: "Do you want to consult/verify something in the Backoffice, or do you need to configure/modify something?"
- Options:
	* `consult` — Only navigate and verify existing configuration. Provide navigation routes
	* `configure` — (DEFAULT) Change values. Provide routes + field-by-field instructions + code equivalent
- Impact:
	* `consult`: Navigation instructions only
	* `configure`: Instructions + equivalent code + impact warnings
- Phase: configure

### DP-2: Which Entity?
- Trigger: User asks to configure something in Backoffice without specifying which entity
- Question: "Which entity do you want to configure? `Repository`, `Application`, `User`, `Role`, `Auth Type`, `Security Policy`, or `other`?"
- Options (each maps to a section file):
	* `repository` — [GAM Backoffice — Repository (GAM_Repository)](./repository.md)
	* `application` — [GAM Backoffice — Applications (GAM_Applications)](./applications.md)
	* `user` — [GAM Backoffice — Users (GAM_Users)](./users.md)
	* `role` — [GAM Backoffice — Roles (GAM_Roles)](./roles.md)
	* `auth_type` — [GAM Backoffice — Authentication Types (GAM_Authentication_Types)](./auth-types.md)
	* `security_policy` — [GAM Backoffice — Security Policies (GAM_Security_Policies)](./security-policies.md)
	* `other` — Request more detail
- Phase: configure

### DP-3: Code Equivalent?
- Trigger: When the user is configuring in Backoffice
- Question: "Do you also want the equivalent GeneXus code to reproduce this configuration by code?"
- Options:
	* `yes` — (DEFAULT) Provide Backoffice instructions + equivalent GeneXus code (refer to [GAM Entity Initialization — Consolidated Pattern](../kb-setup/init/entity-initialization-consolidated.md))
	* `no` — Backoffice instructions only
- Impact: If `yes`, include the code-to-Backoffice mapping from this file
- Phase: configure
