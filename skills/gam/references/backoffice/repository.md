---
name: backoffice-repository
description: GAM Backoffice — Repository settings (General, User Identification, Recovery, Block, Remember Me, Required Data, Session & Security, Timeouts, Integrated Security, Advanced, Emails/SMTP)
---

# GAM Backoffice — Repository (GAM_Repository)
Screen (object name as shipped in `GAM_Web-Administration`): `GAMExampleWWRepositories`

Field mappings verified against GAM Backoffice source. All properties in a single extended form

See also [Object Structure](../kb-setup/init/entity-initialization-consolidated.md#object-structure) for code templates

## Section: General
- Name
	* API Property (`GAMRepository`): `.Name`
- Description
	* API Property (`GAMRepository`): `.Description`
- Default Authentication Type
	* API Property (`GAMRepository`): `.DefaultAuthenticationTypeName`
- Default Role
	* API Property (`GAMRepository`): `.DefaultRoleId`
- Cache Timeout
	* API Property (`GAMRepository`): `.CacheTimeout`
	* Notes: Minutes

## Section: User Identification
- Identification Type
	* API Property (`GAMRepository`): `.IdentificationType`
	* Notes: Login by username/email/both
- User Name Format
	* API Property (`GAMRepository`): `.UserNameFormat`
	* Notes: Email format, free text, etc

## Section: Recovery Password
- Enable Recovery by Email
	* API Property (`GAMRepository`): `.RecoveryPasswordByEmail`
- Recovery Email Object
	* API Property (`GAMRepository`): `.RecoveryPasswordEmailObject`

## Section: Block Access
- Login Attempts to Lock
	* API Property (`GAMRepository`): `.LoginAttemptsToLock`
- Auto Unlock Time
	* API Property (`GAMRepository`): `.AutoUnlockTime`
	* Notes: Minutes; 0 = manual unlock only

## Section: Remember Me
- Enable Remember Me
	* API Property (`GAMRepository`): `.RememberMeEnable`
- Remember Me Timeout
	* API Property (`GAMRepository`): `.RememberMeTimeout`
	* Notes: Days

## Section: Required Data
- Email Required
	* API Property (`GAMRepository`): `.EmailRequired`
- First Name Required
	* API Property (`GAMRepository`): `.FirstNameRequired`
- Last Name Required
	* API Property (`GAMRepository`): `.LastNameRequired`
- Birthday Required
	* API Property (`GAMRepository`): `.BirthdayRequired`
- Gender Required
	* API Property (`GAMRepository`): `.GenderRequired`

## Section: Session & Security
- Session Statistics
	* API Property (`GAMRepository`): `.SessionStatistics`
- Allow Anonymous Sessions
	* API Property (`GAMRepository`): `.AllowAnonymousSessions`
- Session Expires on IP Change
	* API Property (`GAMRepository`): `.SessionExpiresOnIPChange`
- Login Attempts to Lock Session
	* API Property (`GAMRepository`): `.LoginAttemptsToLockSession`
- Minimum Characters in Login
	* API Property (`GAMRepository`): `.MinimumAmountCharactersInLogin`

## Section: Timeouts
BUG (LABEL SWAP): There is a confirmed label swap bug in the Backoffice UI for this section. The label "Timeout for user change password after login" is wired to the API property `TimeoutToCompleteRequiredUserDataAfterLogin`, and the label "Timeout to complete required user data after login" is wired to `TimeoutForUserChangePasswordAfterLogin`. The list below shows the correct API property for each label as it actually appears in the Backoffice. When setting values by code, use the API property name (which is correct); when reading values from the Backoffice UI, be aware the labels are swapped

- Timeout Client Secrets
	* API Property (`GAMRepository`): `.TimeoutClientSecrets`
	* Notes: Minutes
- Timeout for user change password after login
	* API Property (`GAMRepository`): `.TimeoutToCompleteRequiredUserDataAfterLogin`
	* Notes: BUG: label is swapped
- Timeout to complete required user data after login
	* API Property (`GAMRepository`): `.TimeoutForUserChangePasswordAfterLogin`
	* Notes: BUG: label is swapped
- Timeout to finish OAuth authentication using IDP
	* API Property (`GAMRepository`): `.TimeoutToFinishOAuthAuthenticationUsingIDP`
	* Notes: Minutes

## Section: Integrated Security
- Domain Enable
	* API Property (`GAMRepository`): `.IntegratedSecurityByDomainEnable`
- Domain Mode
	* API Property (`GAMRepository`): `.IntegratedSecurityByDomainMode`
- JWT Secret Key
	* API Property (`GAMRepository`): `.JWTSecretKey`
- Encryption Key
	* API Property (`GAMRepository`): `.EncryptionKey`

## Section: Advanced
- Enable SSO REST for Undefined Client IDs
	* API Property (`GAMRepository`): `.EnableSSORESTAccessForUndefinedClientIDs`
	* Notes: Required when Client B's ClientId is not pre-registered in the IDP Application table — see [SSO REST — Single Sign-On for REST Services](../authentication/sso-rest/domain-sso-rest.md)
- Enable Reusing Active User Tokens
	* API Property (`GAMRepository`): `.EnableReusingActiveUserTokens`
- TOTP Secret Key Length
	* API Property (`GAMRepository`): `.TOTPSecretKeyLength`
	* Notes: For OTP/2FA

## Tab: Emails (GAM_Emails)
Email/SMTP is configured **inside the repository** — Backoffice: Repositories → select repository → **Emails** tab (form `GAMExampleWWRepositories`, shipped in `GAM_Web-Administration`)

All values are set via `&GAMRepository.Email.*` sub-properties and persisted with `&GAMRepository.Save()`
See [GAMRepository Code Provider](../kb-setup/init/code-provider/gamrepository-code-provider.md) for the code pattern

> **SMTP is NOT in `client.cfg`** — it is a repository-level setting stored in the GAM database and configurable both from the Backoffice and from code

### Group: Email Configuration (SMTP Server)
- Server Host
	* API Property: `&GAMRepository.Email.ServerHost`
- Server Port
	* API Property: `&GAMRepository.Email.ServerPort`
- Timeout
	* API Property: `&GAMRepository.Email.ServerTimeout`
	* Notes: Seconds
- Secure (SSL/TLS)
	* API Property: `&GAMRepository.Email.ServerSecure`
	* Notes: Boolean
- Sender Email Address
	* API Property: `&GAMRepository.Email.ServerSenderAddress`
- Sender Display Name
	* API Property: `&GAMRepository.Email.ServerSenderName`
- Server Requires Authentication
	* API Property: `&GAMRepository.Email.ServerUsesAuthentication`
	* Notes: Boolean. When False, the two fields below are hidden in the Backoffice and ignored
- Authentication Username (visible only when auth enabled)
	* API Property: `&GAMRepository.Email.ServerAuthenticationUserName`
- Authentication Password (visible only when auth enabled)
	* API Property: `&GAMRepository.Email.ServerAuthenticationUserPassword`
	* Notes: Password field in Backoffice (masked). For Gmail, use an App Password, not the account password

### Group: Account Activation
- Send email when user activates account
	* API Property: `&GAMRepository.Email.SendEmailWhenUserActivateAccount`
	* Notes: Boolean
- Email Subject (shown when enabled)
	* API Property: `&GAMRepository.Email.SubjectWhenUserActivateAccount`
	* Notes: Placeholders — `%1` = application name, `%2` = activation link URL
- Email Body (shown when enabled)
	* API Property: `&GAMRepository.Email.BodyWhenUserActivateAccount`
	* Notes: Same placeholders

### Group: Change Password
- Send email when user changes password
	* API Property: `&GAMRepository.Email.SendEmailWhenUserChangePassword`
	* Notes: Boolean
- Email Subject (shown when enabled)
	* API Property: `&GAMRepository.Email.SubjectWhenUserChangePassword`
	* Notes: `%1` = application name
- Email Body (shown when enabled)
	* API Property: `&GAMRepository.Email.BodyWhenUserChangePassword`

### Group: Change Email / Username
- Send email when user changes email or username
	* API Property: `&GAMRepository.Email.SendEmailWhenUserChangeEmail`
	* Notes: Boolean
- Email Subject (shown when enabled)
	* API Property: `&GAMRepository.Email.SubjectWhenUserChangeEmail`
	* Notes: `%1` = app name
- Email Body (shown when enabled)
	* API Property: `&GAMRepository.Email.BodyWhenUserChangeEmail`
	* Notes: `%1` = app name, `%2` = old email, `%3` = new email

### Group: Password Recovery
- Send email for password recovery
	* API Property: `&GAMRepository.Email.SendEmailToRecoverUserPassword`
	* Notes: Boolean
- Email Subject (shown when enabled)
	* API Property: `&GAMRepository.Email.SubjectToRecoverUserPassword`
	* Notes: `%1` = app name
- Email Body (shown when enabled)
	* API Property: `&GAMRepository.Email.BodyToRecoverUserPassword`
	* Notes: `%1` = app name, `%2` = recovery link URL
