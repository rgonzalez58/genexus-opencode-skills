---
name: backoffice-auth-types
description: GAM Backoffice — Authentication Types (OAuth 2.0 complete field mapping across 6 tabs)
---

# GAM Backoffice — Authentication Types (GAM_Authentication_Types)
Screen (object name as shipped in `GAM_Web-Administration`): `GAMExampleWWAuthTypes`

The form changes dynamically based on the selected type (Apple, Facebook, OAuth 2.0, SAML, etc.)

See also [GAM Authentication Configuration Reference](../authentication/external-providers/common/domain-auth-config.md) for all auth type configurations

## OAuth 2.0 Type — Complete Field Mapping
Field mappings verified against GAM Backoffice source. Main panel (4 tabs) + additional config panel (2 tabs). Total: 6 tabs

### Tab 1: General
- Name
	* API Property (`GAMAuthenticationTypeOAuth20`): `.Name`
	* Notes: Unique auth type name
- Enabled
	* API Property (`GAMAuthenticationTypeOAuth20`): `.IsEnable`
- Function
	* API Property (`GAMAuthenticationTypeOAuth20`): `.FunctionId`
	* Notes: Via `GAMAuthenticationFunctions` enum
- Description
	* API Property (`GAMAuthenticationTypeOAuth20`): `.Description`
- Small Image Name
	* API Property (`GAMAuthenticationTypeOAuth20`): `.SmallImageName`
	* Notes: Icon for login button
- Big Image Name
	* API Property (`GAMAuthenticationTypeOAuth20`): `.BigImageName`
- Impersonate
	* API Property (`GAMAuthenticationTypeOAuth20`): `.Impersonate`
- Client ID Tag
	* API Property (`GAMAuthenticationTypeOAuth20`): `.OAuth20.ClientId_Name`
	* Notes: JSON key name for client_id
- Client ID Value
	* API Property (`GAMAuthenticationTypeOAuth20`): `.OAuth20.ClientId_Value`
	* Notes: Actual client_id value
- Client Secret Tag
	* API Property (`GAMAuthenticationTypeOAuth20`): `.OAuth20.ClientSecret_Name`
	* Notes: JSON key name for client_secret
- Client Secret Value
	* API Property (`GAMAuthenticationTypeOAuth20`): `.OAuth20.ClientSecret_Value`
	* Notes: Actual secret value
- Redirect URL Tag
	* API Property (`GAMAuthenticationTypeOAuth20`): `.OAuth20.RedirectURL_Name`
- Redirect URL Value
	* API Property (`GAMAuthenticationTypeOAuth20`): `.OAuth20.RedirectURL_Value`
- Redirect URL is Custom
	* API Property (`GAMAuthenticationTypeOAuth20`): `.OAuth20.RedirectURL_isCustom`
- Autocomplete Virtual Dir
	* API Property (`GAMAuthenticationTypeOAuth20`): `.OAuth20.RedirectURL_AutocompleteVirtualDirectory`
- Redirect to Authenticate
	* API Property (`GAMAuthenticationTypeOAuth20`): `.OAuth20.RedirectToAuthenticate`

### Tab 2: Authorize
- URL
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.URL`
	* Notes: Authorization endpoint
- Response Type Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.ResponseType_Include`
- Response Type Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.ResponseType_Name`
- Response Type Value
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.ResponseType_Value`
	* Notes: Usually `code`
- Scope Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.Scope_Include`
- Scope Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.Scope_Name`
- Scope Value
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.Scope_Value`
	* Notes: e.g. `openid email profile`
- State Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.State_Include`
- State Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.State_Name`
- Client ID Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.ClientId_Include`
- Client Secret Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.ClientSecret_Include`
- Redirect URL Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.RedirectURL_Include`
- Additional Parameters
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.AdditionalParameters`
	* Notes: Free-form key=value pairs
- Additional Parameters SD
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.AdditionalParametersNativeSD`
	* Notes: SDT for native apps
- Response Access Code Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.ResponseAccessCode_Name`
- Response Error Desc Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Authorize`): `.ResponseErrorDescription_Name`

PKCE sub-section (within Authorize):

- Enable
	* API Property (`…Authorize.PKCEAuthentication`): `.Enable`
- Challenge Length
	* API Property (`…Authorize.PKCEAuthentication`): `.LenghtChallenge`
	* Notes: Note: typo `Lenght` is in the actual API
- Method
	* API Property (`…Authorize.PKCEAuthentication`): `.Method`
	* Notes: S256 / plain

OpenID Connect sub-section (within Authorize):

- Enable
	* API Property (`…Authorize.OpenIDConnectAuthentication`): `.Enable`
- Valid ID Token
	* API Property (`…Authorize.OpenIDConnectAuthentication`): `.ValidIDToken`
- Issuer URL
	* API Property (`…Authorize.OpenIDConnectAuthentication`): `.IssuerURL`
- Use Discovery URL
	* API Property (`…Authorize.OpenIDConnectAuthentication`): `.UseDiscoveryURL`
- Discovery URL
	* API Property (`…Authorize.OpenIDConnectAuthentication`): `.DiscoveryURL`
- Certificate Path
	* API Property (`…Authorize.OpenIDConnectAuthentication`): `.CertificatePathFileName`
- Allow Only Verified Email
	* API Property (`…Authorize.OpenIDConnectAuthentication`): `.AllowOnlyUserEmailVerified`

### Tab 3: Token
- URL
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.URL`
	* Notes: Token endpoint
- Method
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.Method`
	* Notes: POST / GET
- Header Key Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.Header_Key`
- Header Key Value
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.Header_Value`
- Header Auth Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.Header_Authentication_Include`
- Header Auth Method
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.Header_Authentication_Method`
- Header Auth Realm
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.Header_Authentication_Realm`
- Header AuthZ Basic Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.Header_AuthorizationBasic_Include`
- Grant Type Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.GrantType_Include`
- Grant Type Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.GrantType_Name`
- Grant Type Value
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.GrantType_Value`
	* Notes: e.g. `authorization_code`
- Access Code Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.AccessCode_Include`
- Client ID Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.ClientId_Include`
- Client Secret Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.ClientSecret_Include`
- Redirect URL Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.RedirectURL_Include`
- Additional Parameters
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.AdditionalParameters`
- Response Access Token Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.ResponseAccessToken_Name`
	* Notes: e.g. `access_token`
- Response Token Type Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.ResponseTokenType_Name`
	* Notes: e.g. `token_type` — Silent Skip if empty
- Response Expires In Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.ResponseExpiresIn_Name`
- Response Scope Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.ResponseScope_Name`
- Response User Id Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.ResponseUserId_Name`
- Response Refresh Token Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.ResponseRefreshToken_Name`
- Response Error Desc Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.ResponseErrorDescription_Name`
- Autovalidate External Token
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.AutovalidateExternalTokenAndRefresh`
- Refresh Token URL
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Token`): `.RefreshToken_URL`

### Tab 4: UserInfo
- URL
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.UserInfo`): `.URL`
	* Notes: UserInfo endpoint
- Method
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.UserInfo`): `.Method`
- Header Key Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.UserInfo`): `.Header_Key`
- Header Key Value
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.UserInfo`): `.Header_Value`
- Header Authorization Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.UserInfo`): `.Header_Authorization_Include`
- Access Token Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.UserInfo`): `.AccessToken_Include`
- Access Token Name
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.UserInfo`): `.AccessToken_Name`
- Client ID Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.UserInfo`): `.ClientId_Include`
- Client ID Name
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.UserInfo`): `.ClientId_Name`
- Client Secret Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.UserInfo`): `.ClientSecret_Include`
- Client Secret Name
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.UserInfo`): `.ClientSecret_Name`
- User ID Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.UserInfo`): `.UserId_Include`
- User ID Name
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.UserInfo`): `.UserID_Name`
	* Notes: Note: inconsistent casing in API
- Additional Parameters
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.UserInfo`): `.AdditionalParameters`

UserInfo Response Mapping:

- User External Id Tag
	* API Property (`…UserInfo.Response*`): `.ResponseUserExternalId_Name`
	* Notes: Key field for user matching
- User Email Tag
	* API Property (`…UserInfo.Response*`): `.ResponseUserEmail_Name`
- User Verified Email Tag
	* API Property (`…UserInfo.Response*`): `.ResponseUserVerifiedEmail_Name`
- User Name Tag
	* API Property (`…UserInfo.Response*`): `.ResponseUserName_Name`
- User First Name Tag
	* API Property (`…UserInfo.Response*`): `.ResponseUserFirstName_Name`
- User Last Name Auto-generate
	* API Property (`…UserInfo.Response*`): `.ResponseUserLastName_GenerateAutomatic`
- User Last Name Tag
	* API Property (`…UserInfo.Response*`): `.ResponseUserLastName_Name`
- User Gender Tag
	* API Property (`…UserInfo.Response*`): `.ResponseUserGender_Name`
- User Gender Values
	* API Property (`…UserInfo.Response*`): `.ResponseUserGender_Values`
	* Notes: Mapping external values
- User Birthday Tag
	* API Property (`…UserInfo.Response*`): `.ResponseUserBirthday_Name`
- User URL Image Tag
	* API Property (`…UserInfo.Response*`): `.ResponseUserURLImage_Name`
- User URL Profile Tag
	* API Property (`…UserInfo.Response*`): `.ResponseUserURLProfile_Name`
- User Language Tag
	* API Property (`…UserInfo.Response*`): `.ResponseUserLanguage_Name`
- User Time Zone Tag
	* API Property (`…UserInfo.Response*`): `.ResponseUserTimeZone_Name`
- Error Description Tag
	* API Property (`…UserInfo.Response*`): `.ResponseErrorDescription_Name`

UserInfo Custom Properties (grid):

- Attribute Name
	* API Property (`…UserInfo.ResponseUserProperties[n]`): `.Name`
	* Notes: Custom claim name
- Attribute Tag
	* API Property (`…UserInfo.ResponseUserProperties[n]`): `.Tag`
	* Notes: JSON key in UserInfo response

### Tab 5: Roles (in AddConf panel)
Located in the Additional Configuration panel, tab "UserRoles"

- URL
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.URL`
	* Notes: External roles endpoint
- Method
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.Method`
- Header Key Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.Header_Key`
- Header Key Value
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.Header_Value`
- Header Authorization Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.Header_Authorization_Include`
- Access Token Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.AccessToken_Include`
- Access Token Name
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.AccessToken_Name`
- Client ID Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.ClientId_Include`
- Client ID Name
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.ClientId_Name`
- Client Secret Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.ClientSecret_Include`
- Client Secret Name
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.ClientSecret_Name`
- User ID Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.UserID_Include`
- User ID Name
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.UserID_Name`
- Include User ID in URL
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.URI_UserID_Include`
- Include User ID in Header
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.Header_UserID_Include`
- Additional Parameters
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.AdditionalParameters`
- Error Description Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.ResponseErrorDescription_Name`
- Role Name Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Roles`): `.ResponseRole_ExternalID_Name`
	* Notes: Maps external role ID to GAM role

Roles Custom Properties (grid):

- Attribute Name
	* API Property (`…Roles.ResponseRoleProperties[n]`): `.Name`
- Attribute Tag
	* API Property (`…Roles.ResponseRoleProperties[n]`): `.Tag`

### Tab 6: Signout (in AddConf panel)
Located in the Additional Configuration panel, tab "Signout"

- SLO Enable
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.SLOEnable`
	* Notes: Master toggle for external IDP signout
- URL
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.URL`
	* Notes: Signout/revoke endpoint
- Method
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.Method`
- Header Key Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.Header_Key`
- Header Key Value
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.Header_Value`
- Header Authorization Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.Header_Authorization_Include`
- Access Token Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.AccessToken_Include`
- Access Token Name
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.AccessToken_Name`
- Client ID Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.ClientId_Include`
- Client ID Name
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.ClientId_Name`
- Client Secret Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.ClientSecret_Include`
- Client Secret Name
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.ClientSecret_Name`
- State Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.State_Include`
- State Name
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.State_Name`
- User ID Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.UserID_Include`
- User ID Name
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.UserID_Name`
- Include User ID in URL
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.URI_UserID_Include`
- Include User ID in Header
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.Header_UserID_Include`
- Redirect URL Include
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.RedirectURL_Include`
- Redirect URL Name
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.RedirectURL_Name`
- Redirect URL Value
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.RedirectURL_Value`
- Additional Parameters
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.AdditionalParameters`
- Error Description Tag
	* API Property (`GAMAuthenticationTypeOAuth20.OAuth20.Signout`): `.ResponseErrorDescription_Name`
