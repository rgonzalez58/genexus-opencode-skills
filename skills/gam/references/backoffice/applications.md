---
name: backoffice-applications
description: GAM Backoffice — Applications section (General, WEB Auth, REST, Access Control, SSO REST, STS, MiniApp, API Key, Environment, SLO Deep Dive, Languages, Permissions)
---

# GAM Backoffice — Applications (GAM_Applications)
Screens (object names as shipped in `GAM_Web-Administration`):

- `GAMExampleWWApplications` — Application list (top-level)
- `GAMExampleWWApplicationsChildren` — Child applications list (hierarchy)
- `GAMExampleWWAppPermissions` — Permissions tab
- `GAMExampleWWAppMenus` — Menus tab
- `GAMExampleWWAppMenuOptions` — Menu options inside a menu

Field mappings verified against GAM Backoffice source. Backoffice sections: General, Configuration > WEB Auth, Configuration > REST, Configuration > Access Control, Configuration > SSO REST, Configuration > STS Protocol, Configuration > MiniApp, Configuration > API Key, Configuration > Environment

## Section: General
- Name
	* API Property (`GAMApplication`): `.Name`
- Description
	* API Property (`GAMApplication`): `.Description`
- Version
	* API Property (`GAMApplication`): `.Version`
- Company
	* API Property (`GAMApplication`): `.CompanyName`
- Copyright
	* API Property (`GAMApplication`): `.Copyright`
- Use Absolute URL by Environment
	* API Property (`GAMApplication`): `.UseAbsoluteUrlByEnvironment`
- Home Object
	* API Property (`GAMApplication`): `.HomeObject`
	* Notes: Entry point object name
- Account Activation Object
	* API Property (`GAMApplication`): `.AccountActivationObject`
- Logout Object
	* API Property (`GAMApplication`): `.LogoutObject`
	* Notes: Object called on logout
- Return Menu Options Without Permission
	* API Property (`GAMApplication`): `.ReturnMenuOptionsWithoutPermission`
- Main Menu Id
	* API Property (`GAMApplication`): `.MainMenuId`
- Is Base Application
	* API Property (`GAMApplication`): `.IsBaseApplication`
- Application Base
	* API Property (`GAMApplication`): `.ApplicationBase`
	* Notes: Parent app reference

## Section: Configuration > WEB Auth
- Client Id
	* API Property (`GAMApplication`): `.ClientId`
	* Notes: OAuth client identifier
- Client Secret
	* API Property (`GAMApplication`): `.ClientSecret`
	* Notes: Read via property; set via `.SetClientSecret()`
- Client Revoked
	* API Property (`GAMApplication`): `.ClientRevoked`
- Allow Remote Authentication
	* API Property (`GAMApplication`): `.ClientAllowRemoteAuthentication`
	* Notes: Master toggle for OAuth
- OAuth 2.0 Option
	* API Property (`GAMApplication`): `.ClientAllowRemoteAuthenticationOAuth20Option`
- PKCE Method
	* API Property (`GAMApplication`): `.ClientAllowOAuth20PKCEMethod`
- User Data scope
	* API Property (`GAMApplication`): `.ClientAllowGetUserData`
- User Additional Data
	* API Property (`GAMApplication`): `.ClientAllowGetUserAdditionalData`
- User Roles
	* API Property (`GAMApplication`): `.ClientAllowGetUserRoles`
- Session Initial Properties
	* API Property (`GAMApplication`): `.ClientAllowGetSessionInitialProperties`
- Session Application Data
	* API Property (`GAMApplication`): `.ClientAllowGetSessionApplicationData`
- Additional Scope
	* API Property (`GAMApplication`): `.ClientAllowAdditionalScope`
- Image URL
	* API Property (`GAMApplication`): `.ClientImageURL`
- Local Login URL
	* API Property (`GAMApplication`): `.ClientLocalLoginURL`
	* Notes: MANDATORY — the Backoffice refuses to save the application without it. Login object URL, OR `auth:<authtype>[;…]` to auto-redirect to OAuth 2.0/GAMRemote auth types (IdP acts as login proxy) — see [IDP→IDP Login Chaining — `auth:` Local Login URL Proxy](../authentication/local-idp/idp-chaining.md). When GAM acts as Web IDP Server, use the shipped `GAMExampleIDPLogin` object — see [GAM as Web IDP Server — Authorization Code Flow](../authentication/local-idp/authorization-code-flow.md)
- Callback URL
	* API Property (`GAMApplication`): `.ClientCallbackURL`
- Custom Callback URL
	* API Property (`GAMApplication`): `.ClientCallbackURLisCustom`
	* Notes: Boolean; if true, uses literal URL
- State Param Name
	* API Property (`GAMApplication`): `.ClientCallbackURLStateName`
- Disable SLO
	* API Property (`GAMApplication`): `.ClientSingleLogoutDisableSLO`
- Custom SLO URLs
	* API Property (`GAMApplication`): `.ClientSingleLogoutCustomURLsSLO`
	* Notes: Semicolon-separated; see SLO Deep Dive below
- Valid URLs after SLO
	* API Property (`GAMApplication`): `.ClientSingleLogoutValidURLsAfterSLO`

## Section: Configuration > REST
- Allow REST v1.0
	* API Property (`GAMApplication`): `.ClientAllowRESTv10Authentication`
	* Notes: Legacy REST auth
- Allow REST
	* API Property (`GAMApplication`): `.ClientAllowRemoteRESTAuthentication`
- + REST scope properties
	* API Property (`GAMApplication`): `.ClientAllowGetUserDataREST`, `.ClientAllowGetUserAdditionalDataREST`, `.ClientAllowGetUserRolesREST`, `.ClientAllowAdditionalScopeREST`
	* Notes: Mirror WEB Auth scopes for REST

## Section: Configuration > Access Control
- Unique Access by User
	* API Property (`GAMApplication`): `.ClientAccessUniqueByUser`
- Access Requires Permission
	* API Property (`GAMApplication`): `.AccessRequiresPermission`
- Delegate Authorization
	* API Property (`GAMApplication`): `.IsAuthorizationDelegated`
- Repository GUID
	* API Property (`GAMApplication`): `.ClientRepositoryGUID`
- Encryption Key
	* API Property (`GAMApplication`): `.ClientEncryptionKey`

## Section: Configuration > SSO REST
> For the conceptual model, end-to-end flow, trace patterns, and error catalog, see [SSO REST — Single Sign-On for REST Services](../authentication/sso-rest/domain-sso-rest.md)

- Enable
	* API Property (`GAMApplication`): `.SSORESTEnable`
- Mode
	* API Property (`GAMApplication`): `.SSORESTMode`
- Auth Type Name
	* API Property (`GAMApplication`): `.SSORESTUserAuthenticationTypeName`
- Server URL
	* API Property (`GAMApplication`): `.SSORESTServerURL`
- Custom URL
	* API Property (`GAMApplication`): `.SSORESTServerURL_isCustom`
- SLO URL
	* API Property (`GAMApplication`): `.SSORESTServerURL_SLO`
- Repository GUID
	* API Property (`GAMApplication`): `.SSORESTServerRepositoryGUID`
- Key
	* API Property (`GAMApplication`): `.SSORESTServerKey`

## Section: Configuration > STS Protocol
- Enable
	* API Property (`GAMApplication`): `.STSProtocolEnable`
- Mode
	* API Property (`GAMApplication`): `.STSMode`
- Authorization User GUID
	* API Property (`GAMApplication`): `.STSAuthorizationUserGUID`
- Password
	* API Property (`GAMApplication`): `.STSServerClientPassword`
- URL
	* API Property (`GAMApplication`): `.STSServerURL`
- Repository GUID
	* API Property (`GAMApplication`): `.STSServerRepositoryGUID`

## Section: Configuration > MiniApp
- Enable
	* API Property (`GAMApplication`): `.MiniAppEnable`
- Mode
	* API Property (`GAMApplication`): `.MiniAppMode`
- Client URL
	* API Property (`GAMApplication`): `.MiniAppClientURL`
- Custom Client
	* API Property (`GAMApplication`): `.MiniAppClientURL_isCustom`
- Client Repository GUID
	* API Property (`GAMApplication`): `.MiniAppClientRepositoryGUID`
- Auth Type Name
	* API Property (`GAMApplication`): `.MiniAppUserAuthenticationTypeName`
- Server URL
	* API Property (`GAMApplication`): `.MiniAppServerURL`
- Custom Server
	* API Property (`GAMApplication`): `.MiniAppServerURL_isCustom`
- Server Repository GUID
	* API Property (`GAMApplication`): `.MiniAppServerRepositoryGUID`

## Section: Configuration > API Key
- Enable
	* API Property (`GAMApplication`): `.APIKeyEnable`
- Timeout (hours)
	* API Property (`GAMApplication`): `.APIKeyTimeout`
- Only Auth Type
	* API Property (`GAMApplication`): `.APIKeyAllowOnlyAuthenticationTypeName`
- Allow Scope Customization
	* API Property (`GAMApplication`): `.APIKeyAllowScopeCustomization`

## Section: Configuration > Environment
- Name
	* API Property (`GAMApplication`): `.Environment.Name`
- HTTPS
	* API Property (`GAMApplication`): `.Environment.SecureProtocol`
- Host
	* API Property (`GAMApplication`): `.Environment.Host`
- Port
	* API Property (`GAMApplication`): `.Environment.Port`
- Virtual Directory
	* API Property (`GAMApplication`): `.Environment.VirtualDirectory`
- Package
	* API Property (`GAMApplication`): `.Environment.ProgramPackage`
	* Notes: Required for Java
- Extension
	* API Property (`GAMApplication`): `.Environment.ProgramExtension`
	* Notes: `.aspx` / empty / etc

## Tab: SLO (Single Logout) — Deep Dive
- Disable SLO
	* API Property: `.ClientSingleLogoutDisableSLO`
- Custom SLO URLs
	* API Property: `.ClientSingleLogoutCustomURLsSLO`
- Valid URLs after SLO
	* API Property: `.ClientSingleLogoutValidURLsAfterSLO`

### `ClientSingleLogoutCustomURLsSLO` — Deep Dive
Purpose: List of SLO handler URLs for clients that do NOT have GAM applied (non-GAM clients). For clients with GAM, the SLO callback URL is always `oauth/gam/callback` and is computed automatically

Format: URLs separated by semicolons (`;`)

```
http://app-d.com/GAM_Dev_ClientDNetCoreSQL/slo;http://other-app.com/MyApp/logout-handler
```

SLO URL selection algorithm: The matching is progressive (app-path → domain) with order-dependent tie-breaking and a 20-iteration safety limit. Full algorithm, key properties, and diagnostic trace pattern: [SLO URL Matching Algorithm](../debugging/common-behavioral-patterns.md#slo-url-matching-algorithm)

What GAM sends to the custom handler:
Only 4 parameters (does NOT send `server_ip`, does NOT send `repository`):
- `client_id` — Application ID
- `redirect_uri` — Callback URL for return (always `oauth/gam/callback` of the sub-IDP)
- `state` — State parameter for correlation (prefix `SLOInt…`)
- `token` — Access token of the session to terminate

Non-GAM handler requirements:
- Receive the parameters via QueryString
- Invalidate the local session (clear WebSession, tokens)
- Auto-redirect to `redirect_uri?state=<state>` — no manual interaction
- The redirect MUST be automatic to avoid breaking the SLO chain

Common errors:

- URL does not match actual deployment
	* Symptom: `AppCliSLOURL` in traces shows incorrect URL
	* Fix: Correct the URL in this field
- Multiple URLs from the same domain
	* Symptom: Algorithm selects the first one by domain
	* Fix: Order specific URLs first, or remove duplicates
- Test/placeholder URLs
	* Symptom: Redirect to nonexistent endpoint (404)
	* Fix: Clean up obsolete URLs
- Handler requires manual click
	* Symptom: SLO chain stops at the non-GAM client
	* Fix: Handler must auto-redirect
- `?&state=` format in return redirect
	* Symptom: State is not parsed correctly
	* Fix: Use `?state=` (without extra `&`)

Diagnostics: See [SLO with Non-GAM Clients](../debugging/common-debugging.md#slo-with-non-gam-clients) for SLO debugging steps with non-GAM clients

Detailed flow reference: See [GAM Single Log Out (SLO)](../logout/domain-slo.md) section "Non-GAM Client SLO"

## Tab: Languages
- Culture
	* API Property (`GAMApplication`): `.Languages[n].Culture`
- Name
	* API Property (`GAMApplication`): `.Languages[n].Name`
- Description
	* API Property (`GAMApplication`): `.Languages[n].Description`
- Online
	* API Property (`GAMApplication`): `.Languages[n].Online`

## Tab: Permissions
Permission management (GAMApplicationPermission) with parent-child hierarchy
