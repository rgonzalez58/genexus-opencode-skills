---
name: domain-authz
description: Authorization and permission evaluation — grant/deny logic, role evaluation, GAMCheckUserPermission
---

# GAM Authorization — Permission Evaluation
Scope of this file: how GAM evaluates permissions and roles at request time

Related files:
- [GAM Application Menu System](domain-menu.md) — GAMApplication menu system and permission-filtered navigation
- [GAMRole Code Provider](../kb-setup/init/code-provider/gamrole-code-provider.md) — permission and role CRUD via External Objects
- [GAMUser Code Provider](../kb-setup/init/code-provider/gamuser-code-provider.md) — user CRUD and role assignment
- [GAM Session and Token Reference](../authentication/external-providers/common/domain-session-token.md) — token validation, session expiration
- [GAM Impersonation](../authentication/external-providers/common/domain-impersonation.md) — impersonation flow
- [GAM Trace Signatures Catalog](../debugging/trace-signatures-catalog.md#authz) — authorization trace signatures

---

## Permission Evaluation Flow
When a request arrives for a protected resource, GAM performs this evaluation:

- Extract token from `Authorization: Bearer` header or session cookie
- Validate session — look up the token in the active sessions table, verify it is active and not expired
- Check application scope — the session must belong to the same Application as the request
- Evaluate permissions — iterate through the user's active roles and their associated permissions, matching by permission name
- Apply grant/deny logic — if any role explicitly denies the permission, that overrides all grants (Explicit Deny wins). If no role grants the permission, it is implicitly denied
- Return result — Grant or Deny, with corresponding error code if denied

## Evaluation Rules
- Implicit Deny
	* Behavior: if a permission is not explicitly granted, it is denied
	* Example: user with no roles → everything denied
- Explicit Deny Override
	* Behavior: if any role has Deny, it wins over Grant
	* Example: Role A grants P1, Role B denies P1 → P1 denied
- Namespace Match
	* Behavior: permission must belong to the Application of the token
	* Example: token from App A does not see permissions from App B
- Active roles only
	* Behavior: only roles with `GAMUsrRolActive = True` are evaluated
	* Example: deactivated role = as if it did not exist

## Error Codes
- Code 20 — `AccessDenied`
	* Cause: permission not granted in any of the user's roles
	* Fix: assign the permission to the role in Backoffice or via [init/permissions.md](../kb-setup/init/code-provider/gamrole-code-provider.md)
- Code 30 — `Unauthenticated`
	* Cause: token not present or completely invalid
	* Fix: login required
- Code 114 — `InvalidToken`
	* Cause: token expired or does not exist in GAMSession
	* Fix: re-login
- Code 17 — `SessionExpired`
	* Cause: `GAMSesExpires < Now()`
	* Fix: re-login, adjust `OauthTokenExpire` (see [GAM Security Policies Encyclopedia](domain-security-policies.md))

## Diagnostic Traces
See [../debugging/trace-signatures-catalog.md#authz](../debugging/trace-signatures-catalog.md#authorization-authz) for the authorization trace set (grant, deny, role evaluation, user state)

## Runtime Queries
Read current user roles and permissions (authenticated session):

```genexus
&Roles = &GAMUser.GetRoles(&Errors)
&Permissions = &GAMUser.GetPermissions(&Errors)
```

Read for any user by GUID:

```genexus
&Roles = GAMRepository.GetUserRoles(&UserGUID, &Errors)
&Permissions = GAMRepository.GetUserPermissions(&UserGUID, &Errors)
```

Set main role for a user:

```genexus
&Ok = &GAMUser.SetMainRoleById(&RoleId, &Errors)
```

For permission and role creation, deletion, and assignment, see [init/permissions.md](../kb-setup/init/code-provider/gamrole-code-provider.md)

---

## Decision Points
### DP-1: Grant style — Roles-only or Role+Permission overrides?
- Trigger: designing authorization policy for a new Application
- Options:
	* `roles-only` — (DEFAULT) permissions inherited exclusively from role assignments. Simpler policy, easier audit
	* `explicit-deny` — leverage Deny flag on role-permission entries to override grants. Use only when a user must lose access to a specific permission while keeping the role
- Impact: `explicit-deny` complicates audit and troubleshooting. Choose `roles-only` unless hierarchical role structure requires exceptions
- Phase: design

### DP-2: Permission granularity
- Trigger: defining permissions for a new application feature
- Options:
	* `by-screen` — (DEFAULT) one permission per screen/menu option
	* `by-action` — one permission per CRUD action (Create, Read, Update, Delete)
	* `by-feature` — one permission per business feature (can combine many screens)
- Impact: finer granularity means more entries to manage and more checks at runtime. Coarser granularity limits fine-grained control
- Phase: design
