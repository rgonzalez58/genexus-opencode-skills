---
name: scopes
description: OAuth 2.0 user scopes supported by GAM — predefined scopes, user-specific scopes, custom attribute scopes, and the identification rule
---

# OAuth User Scopes
Official documentation: [GAM - OAuth User Scopes](https://docs.genexus.com/en/wiki?55603)

Related files:
- [GAM as Web IDP Server — Authorization Code Flow](authorization-code-flow.md) — Web IDP endpoints (Authorization Code) where scopes are used
- [GAM as REST IDP Server — Password Grant Flow](password-grant-flow.md) — REST IDP endpoints (Password Grant) where scopes are used
- [GAM Authentication](../external-providers/common/domain-auth.md) — Internal GAMRemote flow (external IDP scope is set per auth type)
- [GAM Backoffice — Applications (GAM_Applications)](../../backoffice/applications.md) — Application-level scope properties

Scopes control which user data is returned in the `userinfo` endpoint response and what information is included in the token response. They apply to both Web IDP and REST IDP flows

## Main Scopes
- `gam_user_data`: returns basic user data (guid, username, email, first_name, last_name, birthday, gender, phone, address, etc.) — general scope, includes most UserInfo fields
- `gam_user_additional_data`: returns dynamic user attributes (`attributes` property in response) — requires `&Application.ClientAllowGetUserAdditionalData = True` in Backoffice
- `gam_user_roles`: returns roles assigned to the user (`roles` property in response) — returns array of role names
- `session_initial_prop`: returns properties set at login (`initial_properties` property in response) — related to "HowTo: Send and receive properties set at login"
- `session_application_data`: returns session application data (`application_data` property in response) — related to `GAMSession.GetApplicationData()` / `SetApplicationData()`
- `fullcontrol`: ALL scopes above combined — equivalent to `gam_user_data+gam_user_additional_data+gam_user_roles+session_initial_prop+session_application_data`

## User-Specific Scopes
When NOT using `gam_user_data` and only specific fields are needed:

- `user_email`: returns user email
- `user_phone`: returns user phone
- `user_guid`: returns user GUID
- `user_username`: returns user username
- `user_external_id`: returns user external ID

## Custom Scopes (User Attributes)
Developers can create custom user attributes. Each attribute generates a scope with the pattern: `user_<AttributeID>`

Examples:
- Salary attribute → `user_Salary` scope
- EmployeeID attribute → `user_EmployeeID` scope
- CompanyID attribute → `user_CompanyID` scope

## Combining Scopes
Scopes are concatenated with `+`:

```
scope=gam_user_data+gam_user_roles+session_initial_prop
```

## Identification Rule — CRITICAL
If `gam_user_data` is NOT included in the scope, at least one of these identification scopes MUST be included:
- `user_guid`
- `user_email`
- `user_username`
- `user_external_id`

If no identification scope is included → GAMError 5: "User identification not valid"

This applies especially in GAMRemote / GAMRemoteREST flows where the IDP needs to identify the user to create the session

## Configuration Properties (Backoffice)
### `ClientDoNotShareUserIDs`
- `&GAMApplication.ClientDoNotShareUserIDs` (Backoffice label: "Do not share user IDs"): prevents transmitting the real user GUID and ExternalID. GAM generates an alternative GUID returned in the `external_id` field

Useful when the IDP does not want to expose its internal user IDs to external applications

### `ClientAuthenticationRequestMustIncludeUserScopes`
- `&GAMApplication.ClientAuthenticationRequestMustIncludeUserScopes` (Backoffice label: "Authentication request must include user scopes?"): when `True`, the access_token request does not require sending the `scope` parameter explicitly — the response includes all scopes enabled for the application

## Scope → Fields in UserInfo Response
- `guid`: requires `gam_user_data` or `user_guid`
- `username`: requires `gam_user_data` or `user_username`
- `email`: requires `gam_user_data` or `user_email`
- `verified_email`: requires `gam_user_data`
- `first_name`: requires `gam_user_data`
- `last_name`: requires `gam_user_data`
- `external_id`: requires `gam_user_data` or `user_external_id`
- `birthday`: requires `gam_user_data`
- `gender`: requires `gam_user_data`
- `url_image`: requires `gam_user_data`
- `url_profile`: requires `gam_user_data`
- `phone`: requires `gam_user_data` or `user_phone`
- `address`, `city`, `state`, `post_code`: requires `gam_user_data`
- `language`, `timezone`: requires `gam_user_data`
- `CustomInfo`: requires `gam_user_data`
- `roles`: requires `gam_user_roles`
- `attributes`: requires `gam_user_additional_data`
- `initial_properties`: requires `session_initial_prop`
- `application_data`: requires `session_application_data`
- Custom fields (`user_Salary`, etc.): requires corresponding `user_<AttributeID>`
