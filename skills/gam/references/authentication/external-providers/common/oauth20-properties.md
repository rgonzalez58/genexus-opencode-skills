---
name: oauth20-properties
description: Complete OAuth 2.0 property tree (AuthenticationOAuth20SDT) — every settable property with type, default, and purpose
---

# OAuth 2.0 Property Tree (AuthenticationOAuth20SDT)
This is the complete property tree for the `AuthenticationOAuth20SDT`. Every property available for OAuth 2.0 configuration is documented here with its type, default, and purpose. Confirm exact names against the `GAMAuthenticationTypeOAuth20` EO before writing code (EO Verification Protocol)

Hub: [GAM Authentication Configuration Reference](domain-auth-config.md). Idempotent Load-or-New + Save/error shape: [Idempotent Save Pattern](../../../debugging/common-genexus-patterns.md#idempotent-save-pattern-load-or-new). Ready-to-use provider templates: `external-providers/oauth20/provider-*.md`

---

## Top-Level (General Tab) — REQUIRED
- `ClientId_Name` (String, REQUIRED): Parameter name for client ID (always `!"client_id"`)
- `ClientId_Value` (String, REQUIRED): Your OAuth client ID from the IDP
- `ClientSecret_Name` (String, REQUIRED): Parameter name for client secret (always `!"client_secret"`)
- `ClientSecret_Value` (String, REQUIRED): Your OAuth client secret from the IDP
- `RedirectURL_Name` (String, REQUIRED): Parameter name for redirect (always `!"redirect_uri"`)
- `RedirectURL_Value` (String, REQUIRED): Your app URL. Code 245 if missing
- `RedirectURL_isCustom` (Boolean, optional): False = GAM appends `/oauth/gam/callback`
- `RedirectURL_AutocompleteVirtualDirectory` (Boolean, optional): True = GAM auto-appends virtual dir
- `RedirectToAuthenticate` (Boolean, optional): True = redirect-based flow (browser login)

WARNING: `RedirectURL_Value` MUST be set. If empty, GAM throws Code 245 with no useful error message. This is the most common misconfiguration

## Authorize Sub-SDT — REQUIRED
All properties in this section configure the OAuth 2.0 authorization request. The `URL` is required; most other properties have sensible defaults

- `URL` (String, REQUIRED): IDP authorization endpoint
- `ResponseType_Include` (Boolean, default True): Include response_type param
- `ResponseType_Name` (String, default `!"response_type"`): Parameter name
- `ResponseType_Value` (String, default `!"code"`): Always "code" for authorization_code flow
- `Scope_Include` (Boolean, default True): Include scope param
- `Scope_Name` (String, default `!"scope"`): Parameter name
- `Scope_Value` (String, varies): Scope string (provider-specific)
- `State_Include` (Boolean, default True): Include state param (CSRF protection)
- `State_Name` (String, default `!"state"`): Parameter name
- `ClientId_Include` (Boolean, default True): Include client_id in authorize URL
- `ClientSecret_Include` (Boolean, default False): NEVER True for authorize
- `RedirectURL_Include` (Boolean, default True): Include redirect_uri
- `AdditionalParameters` (String, default `!""`): Extra params as `key=value&key2=value2`
- `AdditionalParametersNativeSD` (String, default `!""`): Extra params for native/SD apps
- `ResponseAccessCode_Name` (String, default `!"code"`): Name of auth code in response
- `ResponseErrorDescription_Name` (String, default `!"error_description"`): Error field name

### PKCE Sub-SDT (inside Authorize) — ADVANCED
Enable only when the IDP requires PKCE (e.g., Twitter/X). Most providers do not require it

- `PKCEAuthentication.Enable` (Boolean, default False): Enable PKCE (required for Twitter/X)
- `PKCEAuthentication.LenghtChallenge` (Numeric, default 0): Challenge length (0 = default 43)
- `PKCEAuthentication.Method` (String, default `!""`): `"S256"` or `"plain"`

### OpenID Connect Sub-SDT (inside Authorize) — RECOMMENDED
Enable when the IDP supports OIDC (Microsoft, Apple). Required for Apple. Recommended for Microsoft v2

- `OpenIDConnectAuthentication.Enable` (Boolean, default False): Enable OIDC
- `OpenIDConnectAuthentication.ValidIDToken` (Boolean, default False): Validate id_token signature
- `OpenIDConnectAuthentication.IssuerURL` (String, default `!""`): Token issuer URL
- `OpenIDConnectAuthentication.UseDiscoveryURL` (Boolean, default False): Use .well-known discovery
- `OpenIDConnectAuthentication.DiscoveryURL` (String, default `!""`): Discovery endpoint URL
- `OpenIDConnectAuthentication.CertificatePathFileName` (String, default `!""`): Path to OIDC cert file
- `OpenIDConnectAuthentication.AllowOnlyUserEmailVerified` (Boolean, default False): Reject unverified emails

See `oidc-certificate-management.md` for how to download and rotate the signing certificate

## Token Sub-SDT — REQUIRED
Configures the token exchange request. The `URL` is required. Header and authentication settings vary by provider

- `URL` (String, REQUIRED): IDP token endpoint
- `Method` (Enum, default `GAMAccessMethod.POST`): HTTP method
- `Header_Key` (String, default `!"Content-type"`): Request header key
- `Header_Value` (String, default `!"application/x-www-form-urlencoded"`): Request header value
- `Header_Authentication_Include` (Boolean, default False): Include HTTP auth header
- `Header_AuthorizationBasic_Include` (Boolean, default False): Include Basic auth header
- `Header_Authentication_Method` (Enum, default `HttpAuthenticationType.Basic`): Auth method type
- `Header_Authentication_Realm` (String, default `!""`): Auth realm
- `GrantType_Include` (Boolean, default True): Include grant_type param
- `GrantType_Name` (String, default `!"grant_type"`): Parameter name
- `GrantType_Value` (String, default `!"authorization_code"`): Grant type value
- `AccessCode_Include` (Boolean, default True): Include authorization code
- `ClientId_Include` (Boolean, default True): Include client_id in token request
- `ClientSecret_Include` (Boolean, default True): Include client_secret in token request
- `RedirectURL_Include` (Boolean, default True): Include redirect_uri in token request
- `AdditionalParameters` (String, default `!""`): Extra params
- `ResponseAccessToken_Name` (String, default `!"access_token"`): Access token field in response
- `ResponseTokenType_Name` (String, default `!"token_type"`): Token type field (Silent Skip if empty)
- `ResponseExpiresIn_Name` (String, default `!"expires_in"`): Expiry field
- `ResponseScope_Name` (String, default `!""`): Scope field in response
- `ResponseUserId_Name` (String, default `!""`): User ID field in response
- `ResponseRefreshToken_Name` (String, default `!"refresh_token"`): Refresh token field
- `ResponseErrorDescription_Name` (String, default `!"error_description"`): Error field
- `AutovalidateExternalTokenAndRefresh` (Boolean, default True): Auto-validate and refresh tokens
- `RefreshToken_URL` (String, default `!""`): Refresh token URL (if different from Token URL)
- `RefreshToken_ClientSecret_Include` (Boolean, default False): Include secret in refresh request

## UserInfo Sub-SDT — REQUIRED (unless OIDC provides user data via ID token)
Configures the user info request. The `URL` is required unless OIDC is enabled and the IDP returns all user data in the ID token (e.g., Apple)

- `URL` (String, REQUIRED): IDP user info endpoint
- `Method` (Enum, default `GAMAccessMethod.GET`): HTTP method
- `Header_Key` (String, default `!"Content-type"`): Request header key
- `Header_Value` (String, default `!"application/json;charset=utf-8"`): Request header value
- `Header_Authorization_Include` (Boolean, default True): Send `Authorization: Bearer` header
- `AccessToken_Include` (Boolean, default False): Send token as query param instead
- `AccessToken_Name` (String, default `!"access_token"`): Query param name for token
- `ClientId_Include` (Boolean, default False): Include client_id
- `ClientId_Name` (String, default `!"client_id"`): Parameter name
- `ClientSecret_Include` (Boolean, default False): Include client_secret
- `ClientSecret_Name` (String, default `!"client_secret"`): Parameter name
- `UserId_Include` (Boolean, default False): Include user ID
- `UserID_Name` (String, default `!""`): User ID param name
- `URI_UserID_Include` (Boolean, default False): Include user ID in URI path
- `Header_UserID_Include` (Boolean, default False): Include user ID in header
- `AdditionalParameters` (String, default `!""`): Extra params

### UserInfo Response Mapping — REQUIRED
Maps IDP response fields to GAM user attributes. At minimum, set `ResponseUserExternalId_Name` and `ResponseUserEmail_Name`

- `ResponseUserExternalId_Name` maps to GAM User External ID: Unique ID from IDP (e.g., `!"sub"`, `!"id"`)
- `ResponseUserEmail_Name` maps to GAM User Email: Email field (e.g., `!"email"`, `!"mail"`)
- `ResponseUserVerifiedEmail_Name` maps to Email verified flag: Verified email field
- `ResponseUserName_Name` maps to GAM User Name: Username/display name
- `ResponseUserFirstName_Name` maps to GAM User First Name: First name field
- `ResponseUserLastName_GenerateAutomatic` maps to Auto-generate last name: Generate from name if not provided
- `ResponseUserLastName_Name` maps to GAM User Last Name: Last name field
- `ResponseUserGender_Name` maps to GAM User Gender: Gender field
- `ResponseUserGender_Values` maps to Gender value mapping: IDP gender values
- `ResponseUserBirthday_Name` maps to GAM User Birthday: Birthday field
- `ResponseUserURLImage_Name` maps to GAM User Image URL: Profile image URL
- `ResponseUserURLProfile_Name` maps to GAM User Profile URL: Profile URL
- `ResponseUserLanguage_Name` maps to GAM User Language: Language field
- `ResponseUserTimeZone_Name` maps to GAM User Timezone: Timezone field
- `ResponseUserProperties` maps to Collection: Custom properties (collection of key-value pairs)
- `ResponseErrorDescription_Name` maps to Error field: Error description field name

## Roles Sub-SDT — ADVANCED
Optional. Configure only when the IDP provides a roles/groups endpoint (e.g., Microsoft Graph App Role Assignments)

- `URL` (String): Roles endpoint URL
- `Method` (Enum): HTTP method
- `Header_Key` (String): Request header key
- `Header_Value` (String): Request header value
- `Header_Authorization_Include` (Boolean): Send Bearer token
- `AccessToken_Include` (Boolean): Include token as param
- `AccessToken_Name` (String): Token param name
- `ClientId_Include` (Boolean): Include client_id
- `ClientId_Name` (String): Parameter name
- `ClientSecret_Include` (Boolean): Include client_secret
- `ClientSecret_Name` (String): Parameter name
- `URI_UserID_Include` (Boolean): Include user ID in URI path
- `Header_UserID_Include` (Boolean): Include user ID in header
- `UserID_Include` (Boolean): Include user ID as param
- `UserID_Name` (String): User ID param name
- `AdditionalParameters` (String): Extra params
- `ResponseErrorDescription_Name` (String): Error field name
- `ResponseRole_ExternalID_Name` (String): Role external ID field
- `ResponseRoleProperties` (Collection): Custom role properties

## Signout Sub-SDT — RECOMMENDED
Configure when you need Single Logout (SLO) to revoke tokens or redirect to the IDP logout endpoint

- `SLOEnable` (Boolean): Enable Single Logout for this auth type
- `URL` (String): IDP logout/revoke endpoint
- `Method` (Enum): HTTP method
- `Header_Key` (String): Request header key
- `Header_Value` (String): Request header value
- `Header_Authorization_Include` (Boolean): Send Bearer token
- `AccessToken_Include` (Boolean): Include token as param
- `AccessToken_Name` (String): Token param name
- `ClientId_Include` (Boolean): Include client_id
- `ClientId_Name` (String): Parameter name
- `ClientSecret_Include` (Boolean): Include client_secret
- `ClientSecret_Name` (String): Parameter name
- `State_Include` (Boolean): Include state param
- `State_Name` (String): State param name
- `URI_UserID_Include` (Boolean): Include user ID in URI
- `Header_UserID_Include` (Boolean): Include user ID in header
- `UserID_Name` (String): User ID param name
- `RedirectURL_Include` (Boolean): Include post-logout redirect
- `RedirectURL_Name` (String): Redirect param name (provider-specific)
- `RedirectURL_Value` (String): Post-logout redirect URL
- `AdditionalParameters` (String): Extra params
- `ResponseErrorDescription_Name` (String): Error field name
