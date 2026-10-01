---
name: common-genexus-patterns
description: GeneXus code patterns for integrating with GAM via External Objects, HttpClient, SDTs, and public APIs
---

# GeneXus Patterns for GAM Integration
PURPOSE: Code patterns that GeneXus developers need when writing applications that interact with GAM via External Objects, HttpClient, and public APIs
- For GeneXus language reference, see Nexa
- For GAM internal behavior, see domain-specific references

---

## EO Discovery (prerequisite for all EO code)
Before writing any code against a GAM External Object, read its live definition in the KB — see
the [EO Discovery Protocol](../global-constraints.md#eo-discovery-protocol-mandatory) in
`global-constraints.md`. That is the single source for the lookup mechanism; it is not repeated here

---

## HttpClient Patterns (GAM-specific behavior only)
For the HttpClient object itself (properties, methods, `Execute`, form-encoding, headers,
authentication types), consult Nexa: `common-extended-type-httpclient.md`

GAM-specific behavior to know when calling GAM endpoints from a non-GAM client (token exchange,
userinfo, logout):

- **Silent Skip guard**: GAM-style guard code adds a header only `If not &field.IsEmpty()`. If the
  field is empty (e.g., an IDP omitted `token_type` in its token response), the header is silently
  not added — no error, no trace. The next call fails with a generic error (see [Silent Skip Detection](#silent-skip-detection) below)
- **Basic Auth at the Token Endpoint**: some IDPs require HTTP Basic Auth (RFC 6749 §2.3.1) instead
  of form-encoded `client_id`/`client_secret` — verify which the target IDP expects

---

## SDT Handling
For SDT mechanics (`FromJson`/`ToJson`, member access, collections), consult Nexa:
`object-structured-data-type.md`, `common-serialization.md`, `common-collections.md`

### GAM SDTs you'll encounter
GAM exposes purpose-built SDTs alongside its EOs. Read each one live before use — same discovery
principle as EOs. Name and role only:

- `GAMExternalAuthenticationInputSDT` — auth/SLO input
- `GAMExternalAuthenticationOutputSDT` — token exchange result
- `GAMUserFilter` / `GAMSessionFilter` — listing filters
- `APIStateClientPar` / `APIStateIDPPar` — OAuth flow state (client-side / IDP-side)
- `AuthenticationSaml20SDT` — SAML configuration
- `GAMOTP2FASDT` — OTP/2FA configuration

---

## Error Handling
GAM propagates errors via `Messages` collections (from `GeneXus.Common`). For iterating collections, consult Nexa: `common-collections.md`

### Idempotent Save Pattern (Load-or-New)
Canonical pattern for creating-or-updating any GAM entity by code (auth types, users, roles, applications, security policies, repositories, event subscriptions). Every domain reference that initializes an entity uses this shape — this is the single source; domain files should link here instead of repeating the full block

```genexus
// --- Idempotent: Load-or-New ---
&Entity.Load(&Name)
If not &Entity.Success()
	&Entity = new()
EndIf

// set properties on &Entity …

&Entity.Save()
If &Entity.Success()
	Commit
Else
	&GAMErrorCollection = &Entity.GetErrors()
	For &GAMError in &GAMErrorCollection
		Msg(Format(!"Save error: %1 (GAM%2)", &GAMError.Message, &GAMError.Code), status)
	EndFor
EndIf
```

- `&Entity` is any GAM EO instance (`GAMUser`, `GAMRole`, `GAMAuthenticationTypeOAuth20`, etc.)
- `Load()` before `= new()` is required even for a brand-new entity: it distinguishes Insert-mode from Update-mode. Skipping `Load()` on an existing name causes a duplicate-key Save error; skipping `= new()` after a failed `Load()` causes Save to attempt an Update with no matching row (Error 42)
- `&GAMErrorCollection` type: `GAMError, GeneXusSecurity` — Collection: True

### Common GAM Error Codes
- Code `10` — `InvalidCredentials`: Invalid credentials. Typical cause: wrong password on local login
- Code `11` — `UserNotActive`: User inactive. Typical cause: user deactivated in GAM Backoffice
- Code `12` — `UserLocked`: User locked. Typical cause: exceeded login attempts
- Code `17` — `SessionExpired`: Session expired. Typical cause: token timed out
- Code `20` — `AccessDenied`: Access denied. Typical cause: missing permission for the resource
- Code `22` — `InvalidRepository`: Invalid repository. Typical cause: namespace does not match any repository
- Code `30` — `ApplicationNotFound`: Application not found. Typical cause: AppId/ClientId does not exist
- Code `114` — `InvalidToken`: Invalid token. Typical cause: token expired, revoked, or malformed
- Code `515` — `OIDCValidationFailed`: OIDC validation failed. Typical cause: invalid JWT signature or incorrect claims
- Code `532` — `UserNotFound`: User not found. Typical cause: GUID does not exist in GAM

---

## Code Conventions
### Non-translatable Strings
In GeneXus, prefix string literals with `!"` to mark them as non-translatable. GAM uses this convention for all internal identifiers, HTTP headers, and trace messages

```genexus
// CORRECT: non-translatable string (will NOT be affected by language settings)
&httpClient.AddHeader(!"Authorization", &Token)
&httpClient.AddVariable(!"grant_type", !"authorization_code")

// INCORRECT: translatable string (could be modified by GeneXus translation engine)
&httpClient.AddHeader("Authorization", &Token)
```

### Enum Dot Notation
GAM enums must be referenced via dot notation. Never use string literals for enum values

```genexus
// CORRECT: dot notation
&LogoutType = GAMLogoutType.LocalLogout        // Value 1
&LogoutType = GAMLogoutType.OAuthLogout        // Value 2
&LogoutType = GAMLogoutType.ExternalIDPLogout  // Value 3

// INCORRECT: string literal — PROHIBITED
// &LogoutType = "LocalLogout"

// Common GAM enums:
GAMAuthenticationTypes.Local
GAMAuthenticationTypes.OAuth20
GAMAuthenticationTypes.SAML20
GAMAuthenticationTypes.OTP

HttpMethod.GET
HttpMethod.POST

MessageTypes.Error
MessageTypes.Warning
MessageTypes.Info
```

---

## Diagnostic Patterns
Behavioral patterns for diagnosing issues when integrating with GAM. These do not require access to GAM internals

### Silent Skip Detection
What it is: When a guard condition (`If not &field.IsEmpty()`) evaluates to false, the guarded code is silently skipped. No error is raised, no trace is emitted. The next operation that depends on the skipped code fails with a generic error

How to detect it:
- Find the trace that should appear AFTER the guard (e.g., `GAMTrace-GAMRemote AddHeader: Authorization:`)
- If the trace is ABSENT in the log, the guard evaluated to false
- Look BACKWARDS in the log for what should have populated that field (e.g., the token response)
- The root cause is typically an empty field in a prior response (e.g., `token_type` missing from the IDP's token response)

Common symptoms:
- Error Code 114 (Invalid Token) after an apparently successful token exchange
- Missing `Authorization` header in outbound requests
- Redirect loops where a step silently fails

### Code-to-Log Correlation
GAM traces follow the pattern: `msg(!"GAMTrace-<Component> <Action>: <Data>", status)`

Correlation rules:
- Trace present = the condition guarding it was true, the code path executed
- Trace absent = the condition was false, the code path was skipped (Silent Skip)
- The absence of an expected trace is stronger diagnostic evidence than an error message

Methodology:
- Identify the expected sequence of traces for the flow (e.g., token exchange -> add header -> userinfo call)
- Find the LAST trace that appears normally
- The NEXT expected trace that is missing points to the exact code path that was skipped
- Cross-reference with the input data at that point to find the root cause

### GAM Logging Conventions
GAM emits traces via two mechanisms:

- `msg()` with `status`: Pattern `msg(!"GAMTrace-<Component> <Action>: <Data>", status)`. When to look: runtime flow traces (most common)
- `Log` External Object: Pattern `Log.Debug(!"message", !"$GAM.<Topic>")`. When to look: structured logging (newer GAM versions)

The `$` prefix in the Log topic produces the literal topic name without the `GeneXusUserLog` prefix

---

## Cross-references to Nexa
For GeneXus language constructs beyond GAM integration, refer to Nexa:

- HttpClient complete reference: `common-extended-type-httpclient.md`
- HttpRequest (handling callbacks): `common-extended-type-httprequest.md`
- SDTs: `object-structured-data-type.md`
- JSON/XML serialization: `common-serialization.md`
- Logging (Log EO): `common-external-object-log.md`
- API Security Level: `properties.md` security related properties
- Functions (Format, UrlEncode): `common-functions.md`
- Data types (Guid, VarChar): `common-data-types.md`
- Collections (Messages): `common-collections.md`
