---
name: backoffice-security-policies
description: GAM Backoffice — Security Policies tab (General, Session, OAuth Token, Password)
---

# GAM Backoffice — Security Policies (GAM_Security_Policies)
Screen (object name as shipped in `GAM_Web-Administration`): `GAMExampleWWSecurityPolicies`

Field mappings verified against GAM Backoffice source

See also [Object Structure](../kb-setup/init/entity-initialization-consolidated.md#object-structure) for code templates

## Section: General
- Id
	* API Property (`GAMSecurityPolicy`): `.Id`
	* Notes: Internal numeric ID
- GUID
	* API Property (`GAMSecurityPolicy`): `.GUID`
	* Notes: Read-only, auto-generated
- Name
	* API Property (`GAMSecurityPolicy`): `.Name`

## Section: Session
- Allow Multiple Concurrent Web Sessions
	* API Property (`GAMSecurityPolicy`): `.AllowMultipleConcurrentWebSessions`
- Web Session Timeout
	* API Property (`GAMSecurityPolicy`): `.WebSessionTimeout`
	* Notes: Minutes

## Section: OAuth Token
- OAuth Token Expire
	* API Property (`GAMSecurityPolicy`): `.OauthTokenExpire`
	* Notes: Minutes. MUST be >= Web Server Session Timeout (see Timeout Conflict pattern)
- OAuth Token Maximum Renovations
	* API Property (`GAMSecurityPolicy`): `.OauthTokenMaximumRenovations`
- OAuth Refresh Token Expire
	* API Property (`GAMSecurityPolicy`): `.OauthRefreshTokenExpire`
	* Notes: Minutes
- OAuth Access Code Expire
	* API Property (`GAMSecurityPolicy`): `.OauthAccessCodeExpire`
	* Notes: Minutes

## Section: Password
- Period Change Password
	* API Property (`GAMSecurityPolicy`): `.PeriodChangePassword`
	* Notes: Days; 0 = never
- Minimum Time to Change Passwords
	* API Property (`GAMSecurityPolicy`): `.MinimumTimeToChangePasswords`
	* Notes: Hours
- Minimum Length Password
	* API Property (`GAMSecurityPolicy`): `.MinimumLengthPassword`
- Minimum Numeric Characters
	* API Property (`GAMSecurityPolicy`): `.MinimumNumericCharactersPassword`
- Minimum UpperCase Characters
	* API Property (`GAMSecurityPolicy`): `.MinimumUpperCaseCharactersPassword`
- Minimum Special Characters
	* API Property (`GAMSecurityPolicy`): `.MinimumSpecialCharactersPassword`
- Maximum Password History Entries
	* API Property (`GAMSecurityPolicy`): `.MaximumPasswordHistoryEntries`
	* Notes: Prevents reuse of N last passwords
