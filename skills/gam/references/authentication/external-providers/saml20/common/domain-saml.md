---
name: domain-saml
description: SAML 2.0 SP and IDP configuration, certificate management, assertion validation, SAML SLO
---

# GAM SAML 2.0 Reference
Scope: modern SAML 2.0 in GAM (all generators, all supported versions)

Legacy Java SAML connector (pre-v18u14) lives in a separate file: [GAM Legacy Java SAML Connector (pre-v18u14)](domain-saml-legacy-java.md). New KBs should ignore that file — it is retained only for diagnosing issues on legacy deployments

Related files:
- [GAM Authentication Configuration Reference](../../common/domain-auth-config.md) — Authentication Type property tree configuration
- [../../../../debugging/trace-signatures-catalog.md#saml](../../../../debugging/trace-signatures-catalog.md#saml-saml) — SAML trace signatures
- [GeneXus Patterns for GAM Integration](../../../../debugging/common-genexus-patterns.md) — HttpClient and SDT handling

---

## SAML Architecture in GAM
GAM implements the Service Provider (SP) role. The external IDP handles authentication

### SP-initiated Flow
```
1. User accesses protected App → GAM detects no session
2. GAM builds AuthnRequest XML → Base64 encode
3. HTTP Redirect to IDP: SamlEndpointLocation?SAMLRequest=<base64>
4. User authenticates at the external IDP
5. IDP builds SAML Response with signed Assertion
6. IDP HTTP POST to GAM ACS: SAMLResponse=<base64>
7. GAM parses Response → validates signature → extracts attributes
8. GAM → auto-provisioning user → create session
9. Redirect to original app
```

### IDP-initiated Flow
```
1. User authenticates at the IDP directly
2. IDP sends SAML Response to GAM ACS without prior AuthnRequest
3. GAM parses and validates → create session
4. Redirect to default app
```

### Routing
When the authentication type is SAML 2.0, GAM routes the request through its SAML handler. The handler determines whether the current step is "Go to IDP" (build AuthnRequest and redirect) or "Return from IDP" (parse SAML Response and create session)

---

## Configuration (GAM Backoffice)
### General Tab
- `ServiceProviderEntityId`
	* Purpose: unique ID of your app registered in the IDP
	* Error if misconfigured: IDP rejects the request
- `IdentityProviderEntityId`
	* Purpose: unique ID of the IDP
	* Error if misconfigured: GAM cannot validate the Issuer of the assertion
- `AssertionConsumerService` (ACS URL)
	* Purpose: URL where the IDP sends assertions
	* Error if misconfigured: IDP rejects with "ACS URL mismatch"
- `Custom redirect URL?`
	* Purpose: allows custom callback URL
	* Error if misconfigured: if False, uses default URL

### Credentials Tab
- `CertificatePathFileName`
	* Purpose: path to the X.509 certificate of the SP
	* Error if misconfigured: GAM cannot sign requests
- `UseIDPCertificateBase64`
	* Purpose: if True, uses inline base64 cert
	* Error if misconfigured: alternative to .cer file
- `IDPCertificateBase64`
	* Purpose: IDP cert in base64
	* Error if misconfigured: signature validation fails

### Signin Tab
- `SamlEndpointLocation`
	* Purpose: SSO URL of the IDP (where to send the AuthnRequest)
- `NameIDPolicyFormat`
	* Purpose: NameID format (emailAddress, persistent, transient)

### Signout Tab (SLO)
- `Enable SLO?`
	* Purpose: activates Single Logout via SAML
- `Single Logout Endpoint`
	* Purpose: SLO URL of the IDP
- `Verificate signature`
	* Purpose: validate signature of LogoutResponse

### User Information Tab
Maps SAML attributes → GAMUser fields via `AuthenticationSaml20UserInfoSDT`

### Roles Tab
Maps SAML role attributes → GAM roles via `AuthenticationSaml20UserRolesSDT`

---

## SDT Structure
```
AuthenticationSaml20SDT
├── ServiceProviderEntityId     (VarChar)
├── IdentityProviderEntityId    (VarChar)
├── SamlEndpointLocation        (VarChar)
├── NameIDPolicyFormat          (VarChar)
├── UseIDPCertificateBase64     (Boolean)
├── IDPCertificateBase64        (VarChar - LongVarChar for large cert)
├── CertificatePathFileName     (VarChar)
├── UserInfo                    (AuthenticationSaml20UserInfoSDT)
└── Roles                       (AuthenticationSaml20UserRolesSDT)
```

---

## Certificates by Generator
The certificate format depends on the GeneXus generator

- .NET Framework
	* SP Keystore: `.pfx` (PKCS#12)
	* IDP Certificate: `.cer` (X.509)
	* Typical path: `\web\certs\sp.pfx`
- .NET Core
	* SP Keystore: `.pfx`
	* IDP Certificate: `.cer`
	* Typical path: `\certs\sp.pfx`
- Java
	* SP Keystore: `.jks` (Java KeyStore)
	* IDP Certificate: `.cer`
	* Typical path: `/WEB-INF/certs/sp.jks`

### Certificate Diagnostics
GAM loads the certificate using the platform-native method: .NET uses PKCS#12 loader (`.pfx` with password), Java uses Java KeyStore loader (`.jks` with password)

If the certificate cannot be loaded, the trace shows:
```
GAMTrace-SAML: Certificate load FAILED: <error detail>
```

Common causes: incorrect path, wrong password, format incompatible with generator

---

## SAML Trace Reference
The full SAML trace catalog is in [../../../../debugging/trace-signatures-catalog.md#saml](../../../../debugging/trace-signatures-catalog.md#saml-saml)

Key markers for quick diagnosis:

- SP → IDP request: `GAMTrace-SAML-Request:<base64EncodedAuthnRequest>`
- IDP → SP response: `GAMTrace-SAML-Response:<base64EncodedSAMLResponse>`
- Assertion OK: `GAMTrace-SAML: Signature validation OK`
- Attributes mapped: `GAMTrace-SAML: Attributes extracted: <attributeList>`
- SLO sent: `GAMTrace-SAML-SLO: LogoutRequest sent to IDP=<sloEndpoint>`

---

## SAML SLO Flow
```
1. User logs out from the App
2. GAM builds LogoutRequest XML with SessionIndex and NameID
3. Base64 encode → Redirect to the IDP SLO Endpoint
4. IDP closes the user session
5. IDP sends LogoutResponse to the GAM SLO callback
6. GAM validates signature (if "Verificate signature" = True)
7. GAM closes local session → redirect to login page
```

### Differences with OAuth SLO
- SAML SLO
	* Protocol: XML with digital signature
	* State management: via SessionIndex in assertion
	* Security: signature validation
- OAuth SLO (GAMRemote)
	* Protocol: HTTP redirect with state param
	* State management: via LoginTmp table
	* Security: state hash validation

---

## Debugging SAML — Systematic Methodology
### Pass 1 — Verify configuration
- SP EntityID matches what is registered in the IDP
- IDP EntityID matches what is configured in GAM
- ACS URL matches between GAM and IDP
- Certificates correct for the generator (.pfx / .jks)
- SamlEndpointLocation is the correct IDP URL

### Pass 2 — Verify traces
- Is there `SAML-Request`? If NO → GAM did not build the AuthnRequest
- Is there `SAML-Response`? If NO → IDP did not send response or ACS URL wrong
- Is there `Assertion Issuer`? If NO → XML parsing failed
- Is there `Signature validation OK`? If NO → IDP certificate incorrect
- Is there `Attributes extracted`? If NO → attribute mapping incorrect

### Pass 3 — If traces are not enough
If the error is not visible in GAM traces, escalate to generated code debugging:
- Identify the failure point by the LAST trace present vs the next expected trace
- Use the `/gx-debug-output` skill to insert targeted debug statements in the generated C#/Java code between those two trace points
- Reproduce and capture variables at the decision points

---

## Error Codes — SAML
- `Assertion validation FAILED`
	* Cause: IDP certificate incorrect or expired
	* Fix: renew the IDP certificate in Backoffice
- `Issuer mismatch`
	* Cause: `IdentityProviderEntityId` ≠ Issuer in the assertion
	* Fix: correct the EntityId in config
- `Response expired`
	* Cause: clock skew between GAM server and IDP
	* Fix: synchronize NTP on both servers
- `ACS URL mismatch`
	* Cause: callback URL misconfigured
	* Fix: verify ACS in GAM and in the IDP
- `Certificate load FAILED`
	* Cause: incorrect path or format incompatible
	* Fix: verify .pfx (NET) vs .jks (Java)
- SAML request never reaches the IDP
	* Cause: `SamlEndpointLocation` incorrect
	* Fix: verify the SSO endpoint URL

---

## Supported IDPs
- Microsoft Entra ID
	* Notes: supports SP-initiated and SLO. Use `login.microsoftonline.com/<tenant>/saml2`
- Okta
	* Notes: supports SP-initiated. ACS URL requires trailing slash in some cases
- SAP
	* Notes: compliant with SAML 2.0 standard
- Agesic (Uruguay)
	* Notes: Uruguayan government. Requires state-specific certificates
- Keycloak
	* Notes: open source. Supports SP-initiated, IDP-initiated, and SLO
- ADFS
	* Notes: on-premises Microsoft. Configuration similar to Entra ID

---

## Decision Points
### DP-1: Role — SP or IDP?
- Trigger: user asks about SAML configuration
- Question: "Does your GAM application act as `SP` (Service Provider — authenticates against an external IDP) or as `IDP` (Identity Provider — provides identity to other apps)?"
- Options:
	* `sp` — (DEFAULT) GAM as SP. Requires configuring: IDP certificate, IDP EntityId, SSO endpoint, ACS URL
	* `idp` — GAM as IDP. Requires configuring: own certificate, own EntityId, metadata URL, client applications
- Impact: completely different configuration. SP: sections 2–4 of this file. IDP: see GAM as IDP references
- Phase: configure

### DP-2: Certificate Format
- Trigger: when configuring SAML certificates
- Question: "What certificate format? `.pfx` (Windows/.NET), `.jks` (Java), or `.crt/.cer` (public)?"
- Options:
	* `.pfx` — (DEFAULT for .NET) PKCS#12 with private key. Password required. Used as SP signing certificate
	* `.jks` — (DEFAULT for Java) Java KeyStore. Password required. Java equivalent of .pfx
	* `.crt/.cer` — IDP public certificate (no private key). Used to validate IDP assertions
- Impact: affects the `CertificateFile`, `CertificatePassword` properties and assertion validation
- Phase: configure

### DP-3: Flow Type
- Trigger: when configuring SAML flow
- Question: "SP-initiated (your app redirects to the IDP) or IDP-initiated (the IDP redirects to your app)?"
- Options:
	* `sp_initiated` — (DEFAULT) user enters your app, gets redirected to IDP for login, returns with assertion
	* `idp_initiated` — user enters the IDP portal, selects your app, arrives with direct assertion
- Impact: SP-initiated requires `SamlEndpointLocation` (IDP SSO URL). IDP-initiated requires the IDP to have your ACS URL registered
- Phase: configure

### DP-4: Which IDP?
- Trigger: when configuring SAML as SP (DP-1 = sp)
- Question: "Which IDP will you federate with? `Entra ID (Azure)`, `Okta`, `Keycloak`, `ADFS`, `AGESIC`, `SAP`, or `other`?"
- Options: each IDP has particularities documented in "Supported IDPs" above
- Impact: each IDP has specific URLs, assertion formats, and quirks
- Phase: configure

### DP-5: Legacy Java Connector (pre-v18u14)
- Trigger: user reports SAML issue in Java + GAM prior to v18u14
- Redirect: load [GAM Legacy Java SAML Connector (pre-v18u14)](domain-saml-legacy-java.md) instead of sections 1–9 above
- New KBs should never match this DP
