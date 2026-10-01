---
name: domain-impersonation
description: Impersonation flow — admin acts as another user with full audit trail
---

# GAM Impersonation
Scope: admin impersonating a target user for support, testing, or incident investigation

Related files:
- [GAM Authorization — Permission Evaluation](../../../authorization/domain-authz.md) — permission evaluation (`GAM_Impersonate` check)
- [GAM Session and Token Reference](domain-session-token.md) — session creation and token structure
- [../../../debugging/trace-signatures-catalog.md#impersonation](../../../debugging/trace-signatures-catalog.md#impersonation-impersonation) — impersonation traces

---

## Flow
```
Admin → GAM.Impersonate(<TargetUserId>, <AdminToken>)
	→ Validate that admin has impersonation permission
	→ Create new session as <TargetUserId>
	→ Return new token (linked to the original admin for auditing)
```

## Requirements
- Admin permission
	* Detail: the admin must have the `GAM_Impersonate` permission
- Same repository
	* Detail: admin and target must be in the same GAM repository
- Valid token
	* Detail: the admin's token must be active and not expired
- Auditing
	* Detail: the impersonated session maintains a reference to the original admin

## Error Codes
- Code 20 — `AccessDenied`
	* Cause: admin lacks `GAM_Impersonate` permission
	* Fix: assign the permission to the admin's role
- Code 532 — `UserNotFound`
	* Cause: target user not found
	* Fix: verify UserId and repository
- Code 114 — `InvalidToken`
	* Cause: admin token invalid or expired
	* Fix: re-login as admin

## Diagnostic Traces
See [../../../debugging/trace-signatures-catalog.md#impersonation](../../../debugging/trace-signatures-catalog.md#impersonation-impersonation)
