---
name: backoffice-roles
description: GAM Backoffice — Roles section (field mappings, permissions, hierarchy)
---

# GAM Backoffice — Roles (GAM_Roles)
Screens (object names as shipped in `GAM_Web-Administration`):

- `GAMExampleWWRoles` — Role list
- `GAMExampleWWRoleRoles` — Child-role assignment (role hierarchy)
- `GAMExampleWWRolePermissions` — Role permissions grid

Field mappings verified against GAM Backoffice source

## Complete Field Mapping
- Id
	* API Property (`GAMRole`): `.Id`
	* Notes: Internal numeric ID
- GUID
	* API Property (`GAMRole`): `.GUID`
	* Notes: Read-only, auto-generated
- Name
	* API Property (`GAMRole`): `.Name`
- Description
	* API Property (`GAMRole`): `.Description`
- External Id
	* API Property (`GAMRole`): `.ExternalId`
	* Notes: For external system mapping
- Security Policy
	* API Property (`GAMRole`): `.SecurityPolicyId`
	* Notes: Override user's default policy
- Parent Role
	* API Property (`GAMRole`): `.AddRole(&parentRole, &errors)`
	* Notes: Role hierarchy

Permissions:
- Allow/Deny per permission: `GAMPermissionAccessType.Allow` / `GAMPermissionAccessType.Deny`
- Explicit Deny always wins over Allow
