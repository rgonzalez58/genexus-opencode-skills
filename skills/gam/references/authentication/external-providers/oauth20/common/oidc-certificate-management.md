---
name: oidc-certificate-management
description: Download, install, and rotate OIDC signing certificates for Google, Microsoft Entra ID, and other IDPs
---

# OIDC Certificate Management
Related files:
- [GAM Authentication Configuration Reference](../../common/domain-auth-config.md) — OpenIDConnect Sub-SDT properties
- [OAuth 2.0 Auth Type — Programmatic Initialization](../provider-generic-oauth20.md) — Google and Microsoft OAuth 2.0/OIDC setup (no dedicated provider file — see Provider Comparison in domain-auth-config.md)

## Why Certificates Rotate
IDPs rotate the signing key used for `id_token` validation periodically (Google rotates roughly weekly; Microsoft rotates roughly every 6 weeks). When the certificate cached by GAM no longer matches the live JWKS, ID token validation fails and login silently drops

Symptoms:
- OIDC `ValidIDToken = True` but login fails with no user-visible error
- Traces show `id_token` validation failure shortly after a previously working deployment

GAM stores the certificate at the path set in `OpenIDConnectAuthentication.CertificatePathFileName`. When `UseDiscoveryURL = True`, GAM can fetch keys dynamically from the JWKS URI; the static path is the fallback and the source of truth when discovery is disabled

## Updating Google OIDC Certificate
Google's JWKS is published at `https://www.googleapis.com/oauth2/v1/certs` (JSON object keyed by `kid`). Each value is a PEM-encoded X.509 certificate

```genexus
&httpClient.Execute(HttpMethod.Get, !"https://www.googleapis.com/oauth2/v1/certs")
&ResultHttpClient = &httpClient.ToString()

// Parse the first certificate from the JSON response
// (Use GeneXus JSON data provider or &JSON variable to iterate keys)
// Pick any valid kid — GAM caches a single certificate
&ResultString = <first certificate PEM>

&File.Source = !"<path>\your-certificate.crt"
&File.Create()
&File.WriteAllText(&ResultString)
```

Set the auth type's `CertificatePathFileName` to `<path>\your-certificate.crt`

## Updating Microsoft Entra ID (Azure) OIDC Certificate
Microsoft publishes JWKS at `https://login.microsoftonline.com/<tenant>/discovery/v2.0/keys` (use `organizations` for multi-tenant, a GUID for single-tenant, or `common` for both work and personal accounts)

```genexus
&httpClient.Execute(HttpMethod.Get, !"https://login.microsoftonline.com/organizations/discovery/v2.0/keys")
&ResultHttpClient = &httpClient.ToString()

// Parse the x5c field from the first key in the "keys" array
// x5c is a Base64-encoded DER certificate chain
// Wrap the x5c value with PEM BEGIN/END markers:
//   -----BEGIN CERTIFICATE-----
//   <x5c value>
//   -----END CERTIFICATE-----
&ResultString = !"-----BEGIN CERTIFICATE-----" + Chr(10) + &X5cValue + Chr(10) + !"-----END CERTIFICATE-----"

&File.Source = !"<path>\azure-certificate.crt"
&File.Create()
&File.WriteAllText(&ResultString)
```

Set the auth type's `CertificatePathFileName` to `<path>\azure-certificate.crt`

## Rotation Strategy
- Schedule a GeneXus procedure (via OS scheduler or a GAM Job) that refreshes the certificate file daily or weekly
- Verify the file was updated (check modification time) before using it
- When `UseDiscoveryURL = True`, rely on discovery and skip static certificate management — the static file is only a backup

## Verifying
After rotating, attempt a login with OIDC `ValidIDToken = True`. Enable GAM tracing (`common-debugging.md`). Traces show the key ID resolved from the JWKS and whether the signature matched
