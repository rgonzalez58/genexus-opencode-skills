---
name: backoffice-users
description: GAM Backoffice — Users section (Identity, Status, OTP, Personal, Preferences, Custom Attributes)
---

# GAM Backoffice — Users (GAM_Users)
GeneXus Screens (object names as shipped in `GAM_Web-Administration`):

- `GAMExampleWWUsers` — Paginated user list with filters
- `GAMExampleWWUserRoles` — User roles tab
- `GAMExampleWWUserPermissions` — User permissions tab
- `GAMExampleWWUserApplications` — User applications tab

Field mappings verified against GAM Backoffice source

## Section: Identity
- GUID
	* API Property (`GAMUser`): `.GUID`
	* Notes: Read-only, auto-generated
- NameSpace
	* API Property (`GAMUser`): `.NameSpace`
	* Notes: Read-only
- Authentication Type
	* API Property (`GAMUser`): `.AuthenticationTypeName`
- Name
	* API Property (`GAMUser`): `.Name`
- Email
	* API Property (`GAMUser`): `.EMail`
- Password
	* API Property (`GAMUser`): `.Password`
	* Notes: Write-only (set on create/update)
- First Name
	* API Property (`GAMUser`): `.FirstName`
- Last Name
	* API Property (`GAMUser`): `.LastName`
- External Id
	* API Property (`GAMUser`): `.ExternalId`
	* Notes: For federated identity mapping
- Phone
	* API Property (`GAMUser`): `.Phone`
- URL Image
	* API Property (`GAMUser`): `.URLImage`
	* Notes: User avatar URL

## Section: Account Status
- Is Active
	* API Property (`GAMUser`): `.IsActive`
- Date Last Authentication
	* API Property (`GAMUser`): `.DateLastAuthentication`
	* Notes: Read-only
- Is Blocked
	* API Property (`GAMUser`): `.IsBlocked`
- Must Change Password
	* API Property (`GAMUser`): `.MustChangePassword`
- Password Never Expires
	* API Property (`GAMUser`): `.PasswordNeverExpires`
- Cannot Change Password
	* API Property (`GAMUser`): `.CannotChangePassword`
- Don't Receive Information
	* API Property (`GAMUser`): `.DontReceiveInformation`
- Is Enabled In Repository
	* API Property (`GAMUser`): `.IsEnabledInRepository`
- Security Policy
	* API Property (`GAMUser`): `.SecurityPolicyId`
- Enable Two Factor Auth
	* API Property (`GAMUser`): `.EnableTwoFactorAuthentication`

## Section: OTP Status (all read-only)
- OTP Number Locked
	* API Property (`GAMUser`): `.OTPNumberLocked`
	* Notes: Read-only
- OTP Last Locked Date
	* API Property (`GAMUser`): `.OTPLastLockedDate`
	* Notes: Read-only
- OTP Daily Number Codes
	* API Property (`GAMUser`): `.OTPDailyNumberCodes`
	* Notes: Read-only
- OTP Last Date Request Code
	* API Property (`GAMUser`): `.OTPLastDateRequestCode`
	* Notes: Read-only

## Section: Personal
- Birthday
	* API Property (`GAMUser`): `.Birthday`
- Gender
	* API Property (`GAMUser`): `.Gender`
- Address
	* API Property (`GAMUser`): `.Address`
- Address 2
	* API Property (`GAMUser`): `.Address2`
- City
	* API Property (`GAMUser`): `.City`
- State
	* API Property (`GAMUser`): `.State`
- Post Code
	* API Property (`GAMUser`): `.PostCode`
- Time Zone
	* API Property (`GAMUser`): `.TimeZone`
- URL Image
	* API Property (`GAMUser`): `.URLImage`
- URL Profile
	* API Property (`GAMUser`): `.URLProfile`

## Section: Preferences
- Language
	* API Property (`GAMUser`): `.Language`
- Theme
	* API Property (`GAMUser`): `.Theme`

Available actions:
- Assign main role: `&GAMUser.SetMainRoleById(&roleId, &errors)`
- Generate API Key: `&GAMUser.GenerateApplicationAPIkey(…)`
- Activate/Deactivate: Toggle `.IsBlocked`
- Delete: `&GAMUser.PhysicalDelete(&errors)`
- Send activation email: `&GAMUser.SendEmailToActivateAccount(&app, &link, &errors)`

User Attributes (sub-tab):
- Custom key-value attributes
- Single value: `{Id, Value}`
- Multi-value: `{Id, [{Id, Value}, …]}`

## User Custom Attributes — External Object API
GAM users support dynamic attributes (custom key-value properties) beyond the fixed identity fields, managed via `GAMUser` methods with `GAMEntityProperty` / `GAMUserAttribute` as descriptors:

- Single-valued: `GetAttribute(<id>, &errors)` (returns the descriptor, read `.Value`), `SetAttribute(&attribute, &errors)` (create or update — build the descriptor with `Id`/`Value` first), `DeleteAttribute(&attribute, &errors)` (needs only `Id` set)
- Multi-valued: `GetMultiValuedAttribute(<id>, <itemKey>, &errors)`, `SetMultiValuedAttribute(<id>, <itemKey>, <value>, &errors)` — item-level read/write
- `Commit` after any successful `Set`/`Delete`

`GAMEntityProperty` descriptor:

- `Id` — Attribute identifier
- `Name` — Attribute display name
- `Value` — Attribute value
- `IsMultiValued` — Whether this attribute supports multiple values

Constraints — Custom Attributes:
- After `SetAttribute()` / `DeleteAttribute()`, call `Commit` on success
- Attribute identifiers must align with the repository's extensibility metadata
- Multi-valued attributes require item-level read/write (use `GetMultiValuedAttribute` / `SetMultiValuedAttribute`)
- Reading all attributes: use `&GAMUser.Attributes` (returns collection of `GAMUserAttribute`)
