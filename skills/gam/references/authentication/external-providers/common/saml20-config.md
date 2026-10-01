---
name: saml20-config
description: Complete SAML 2.0 property tree (AuthenticationSAML20SDT) for programmatic auth type configuration
---

# SAML 2.0 Configuration (AuthenticationTypeSAML20SDT)
Hub: [GAM Authentication Configuration Reference](domain-auth-config.md). For in-depth SAML documentation (SP/IDP roles, metadata, attribute mapping, signing), see `domain-saml.md`. For the Microsoft Entra ID SAML variant, see `../saml20/common/domain-saml.md` (Supported IDPs → Microsoft Entra ID)

Variable: `exo:GAMAuthenticationTypeSAML20, GeneXusSecurity`

---

## Complete SAML 2.0 Property Tree — REQUIRED and OPTIONAL
- `ServiceProviderEntityId` (String, REQUIRED): SP Entity ID (must match IDP config)
- `IdentityProviderEntityId` (String, REQUIRED): IDP Entity ID
- `LocalSiteURL` (String, REQUIRED): Your app base URL
- `LocalSiteURL_isCustom` (Boolean, optional): False = auto-detect
- `LocalSiteURL_AutocompleteVirtualDirectory` (Boolean, optional): True = auto-append virtual dir
- `SamlEndpointLocation` (String, REQUIRED): IDP SSO endpoint URL
- `NameIDPolicyFormat` (Enum, optional): `GAMSAML20NameIdPolicyFormats.EMAIL` / `.UNSPECIFIED` / etc
- `ForceAuthn` (Boolean, optional): Force re-authentication
- `AuthnContext` (String, optional): Authentication context class
- SP Signing (Request Signature):
	* `UsePrivateKeyBase64` (Boolean, optional): Use inline Base64 key
	* `PrivateKeyBase64` (String, optional): Base64-encoded private key
	* `RequestSignatureHashAlgorithm` (String, optional): Hash algorithm for signing
	* `KeyStPathCredential` (String, optional): Path to keystore file (.pfx/.jks)
	* `KeyStPwdCredential` (String, optional): Keystore password
	* `KeyAliasCredential` (String, optional): Key alias in keystore
- IDP Certificate (Response Validation):
	* `UseIDPCertificateBase64` (Boolean, optional): Use inline Base64 cert
	* `IDPCertificateBase64` (String, optional): Base64-encoded IDP cert
	* `KeyStoreFilePathTrustCred` (String, optional): Path to trust keystore
	* `KeyStorePwdTrustCred` (String, optional): Trust keystore password
	* `KeyAliasTrustCred` (String, optional): Trust key alias
- UserInfo Mapping:
	* `UserInfo.ResponseUserExternalId_Name` (String, optional): Attribute for external ID
	* `UserInfo.ResponseUserEmail_Name` (String, optional): Attribute for email
	* `UserInfo.ResponseUserName_Name` (String, optional): Attribute for username
	* `UserInfo.ResponseUserFirstName_Name` (String, optional): Attribute for first name
	* `UserInfo.ResponseUserLastName_Name` (String, optional): Attribute for last name
- Roles Mapping:
	* `Roles.ResponseRole_ExternalID_Name` (String, optional): Attribute for role external ID
- SLO (Single Logout):
	* `SLOEnable` (Boolean, optional): Enable SAML SLO
	* `SingleLogoutEndpoint` (String, optional): IDP SLO endpoint URL
	* `SLOVerificateSignature` (Boolean, optional): Verify SLO response signature

## SAML 2.0 Initialization
Idempotent Load-or-New + Save/error shape — see [Idempotent Save Pattern](../../../debugging/common-genexus-patterns.md#idempotent-save-pattern-load-or-new). Set the properties listed above on `&GAMAuthSAML20` plus top-level `Name`, `IsEnable`, `Description`, `Impersonate`, `SmallImageName`. Confirm exact names against the EO (EO Verification Protocol)
