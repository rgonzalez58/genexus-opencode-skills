---
name: domain-saml-legacy-java
description: Legacy Java SAML connector (pre-v18u14) — artech.security.saml.jar. Retained for legacy deployments only; new KBs must ignore this file
---

# GAM Legacy Java SAML Connector (pre-v18u14)
Applies to: Java generator, GAM v17uX and v18u1 through v18u13HF. Do not load this file for new KBs. In v18u14 the SAML implementation was rewritten and `artech.security.saml.jar` is no longer used — use [GAM SAML 2.0 Reference](domain-saml.md) instead

Related files:
- [GAM SAML 2.0 Reference](domain-saml.md) — modern SAML (applies from v18u14 onwards, all generators)
- [GeneXus Environment: Java](../../../../environments/environment-java.md) — Java deployment specifics

---

## Architecture Overview
In this version range, SAML for Java is handled by a separate connector JAR that runs as Tomcat servlets alongside the GeneXus application. GAM API communicates with the connector via a GeneXus procedure call; the connector is NOT part of the GAM API binary

### Component Map
- `artech.security.saml.jar`
	* SAML connector. Deployed in `WEB-INF/lib/`. Contains all SAML servlets and logic
- `artech.security.saml.servlet.SSO`
	* Handles login: `/saml/gam/signin`
- `artech.security.saml.servlet.LOGOUT`
	* Handles logout: `/saml/gam/signout`
- `artech.security.saml.SAMLReceiver`
	* Parses inbound SAML messages (SAMLResponse, LogoutRequest, LogoutResponse)
- `artech.security.saml.SAMLHelper`
	* Builds and sends outbound SAML messages (AuthnRequest, LogoutRequest)
- `artech.security.saml.Propiedades`
	* Loads SAML configuration — either from GAM API or from classpath `.properties` files
- `artech.security.saml.Crypt`
	* Encrypt/decrypt connector secrets (KeyCrypt)
- `artech.security.saml.SAMLBootstrap`
	* Initializes the OpenSAML library on startup
- `gamexternalauthenticationinputsaml20`
	* GeneXus procedure (generated code). Entry point called by the connector to interact with GAM API
- `GetAuthenticationParmsFromState`
	* GAM API procedure called by `Propiedades.loadPropsFromGAM()` to fetch SAML config for a given state

### Required web.xml mappings
The following servlet mappings MUST exist in `WEB-INF/web.xml`:

```xml
<servlet>
	<servlet-name>GAMSaml20SignIn</servlet-name>
	<servlet-class>artech.security.saml.servlet.SSO</servlet-class>
</servlet>
<servlet>
	<servlet-name>GAMSaml20SignOut</servlet-name>
	<servlet-class>artech.security.saml.servlet.LOGOUT</servlet-class>
</servlet>

<servlet-mapping>
	<servlet-name>GAMSaml20SignIn</servlet-name>
	<url-pattern>/saml/gam/signin</url-pattern>
</servlet-mapping>
<servlet-mapping>
	<servlet-name>GAMSaml20SignOut</servlet-name>
	<url-pattern>/saml/gam/signout</url-pattern>
</servlet-mapping>
```

---

## Login Flow
```
1. User accesses protected resource → GAM detects no session
2. GAM calls GAMExternalAuthenticationSaml20
	 → generates state token: SMLSTD<hex>  (e.g. SMLSTD6de9978ace0e4b1995…)
	 → stores state in GAMLoginTmp table
	 → redirects browser to: /saml/gam/signin?RelayState=SMLSTD…

3. SSO servlet receives GET /saml/gam/signin?RelayState=SMLSTD…
	 → Propiedades.init(RelayState)
		 → state != null → loadPropsFromGAM(state)
			 → calls GetAuthenticationParmsFromState(state)  [GAM API]
			 → receives Properties: SamlEndpointLocation, NameIDPolicyFormat,
				 AssertionConsumerServiceURL, SingleLogoutLocation,
				 KeyCrypt, DisableSingleLogout
	 → SAMLHelper.doAuthenticationRedirect()
		 → builds AuthnRequest XML
		 → Base64-encodes + DEFLATE-compresses (HTTP Redirect Binding outbound)
		 → redirects browser to SamlEndpointLocation?SAMLRequest=<B64(DEFLATE(XML))>

4. IDP authenticates user → sends SAMLResponse via HTTP POST to /saml/gam/signin
	 → POST body: SAMLResponse=Base64(XML)   (no DEFLATE — POST Binding)

5. SSO servlet receives POST /saml/gam/signin
	 → SAMLReceiver.getSAMLAssertion(SAMLResponse)
		 → Base64.decode(SAMLResponse) → XML bytes
		 → DocumentBuilder.parse → validates signature
	 → SAMLReceiver.getDataFromAssertion()
		 → reads NameID, attributes (via attProps.properties mapping)
	 → calls gamexternalauthenticationinputsaml20.executeUdp("saml=signin", state, …)
	 → GAM creates session → redirects user to app
```

### State format
The connector uses `RelayState` to carry the GAM state. Two formats:

- `SMLSTD<hex>`
	* Path in GetAuthenticationParmsFromState: ELSE branch → loads SAML config from GAMLoginTmp
	* Used for: SP-initiated login
- `Token=NameIDFormat,NameIDValue::SessionIndex`
	* Path in GetAuthenticationParmsFromState: IF branch → OAuth path (not SAML)
	* Used for: IDP-initiated SLO back-channel

GAMTrace evidence of correct state routing: `GAMTrace-&State NOT startsWith token` — appears when state is `SMLSTD…` (expected, normal)

---

## Logout Flow
### Branch 1: SP-initiated SLO (normal path)
```
1. User clicks logout in the app
2. GAM Logout procedure builds ExternalLogoutURL:
	 /saml/gam/signout?guid=<sessionGUID>&index=<sessionIndex>
		 &nameid=<NameIDFormat,NameIDValue>&RelayState=SMLSTD<hex>
	 GAMTrace: "GAMTrace-Logout &ExternalLogoutURL: …"
3. Browser → GET /saml/gam/signout?guid=…&index=…&nameid=…&RelayState=…

4. LOGOUT.doGet() → doPost()
	 → Propiedades.init(RelayState)  [loads SAML config via GetAuthenticationParmsFromState]
	 → samlParameter(="SAMLResponse") == null AND SAMLRequest == null
	 → crearSAMLlogout(request, response)
		 → reads: guid, index, nameid params from request
		 → splits nameid on ',' → gets NameIDFormat and NameIDValue
		 → SAMLHelper.doLogoutGlobalRedirect(response, index, nameidValue, NameIDFormat)
			 → builds LogoutRequest XML with SessionIndex + NameID
			 → HTTP Redirect Binding outbound: redirects browser to IDP SLO endpoint
				 with SAMLRequest=<B64(DEFLATE(XML))>&SigAlg=…&Signature=…

5. IDP receives LogoutRequest → invalidates session
	 → sends LogoutResponse back to SP
```

### Branch 2: SP receives LogoutResponse from IDP
```
6. IDP → POST /saml/gam/signout  (HTTP POST Binding)
	 → body: SAMLResponse=Base64(XML)

7. LOGOUT.doPost()
	 → samlParameter = request.getParameter("SAMLResponse") = Base64(XML)
	 → samlParameter != null → Branch 2 (LogoutResponse path)
	 → SAMLReceiver.getLogoutResponse(samlParameter, signature, sigAlg)
		 → Base64.decode → XML parse → validates signature
	 → checks status: urn:oasis:names:tc:SAML:2.0:status:Success or PartialLogout
	 → if valid: validarLogoutResponse() → logout()
		 → gamexternalauthenticationinputsaml20.executeUdp("saml=signout", state, …)
		 → GAM closes local session → HTTP 301 redirect to post-logout URL
```

### Branch 3: IDP-initiated SLO (IDP sends LogoutRequest to SP)
```
IDP → GET /saml/gam/signout?SAMLRequest=<B64(DEFLATE(XML))>&SigAlg=…&Signature=…

LOGOUT.doPost()
→ SAMLRequest != null → Branch 3
→ SAMLReceiver.getLogoutRequest(SAMLRequest)
	→ Base64.decode(SAMLRequest) → raw DEFLATE bytes  ← connector expects XML here
	→ DocumentBuilder.parse(DEFLATE bytes) → SAXException [Content not allowed in prolog]
→ GXCSAMLException → logger.error() [silenced if SLF4J NOP active]
→ return  (blank response, no HTTP body)

IDP → POST /saml/gam/signout  (SAMLRequest=Base64(XML) — POST Binding) → WORKS
→ SAMLReceiver.getLogoutRequest(SAMLRequest)
	→ Base64.decode → XML bytes → DocumentBuilder.parse → OK
	→ extracts SessionIndex, NameID → builds Token=NameIDFormat,NameIDValue::SessionIndex
	→ Propiedades.updatePropsFromGAM(Token=…)
	→ validates signature → if valid: SAMLHelper.doLogoutLocalRedirect(…)
```

### Supported bindings
- SP → IDP AuthnRequest (login): HTTP Redirect Binding supported
- SP → IDP LogoutRequest (SLO): HTTP Redirect Binding supported
- IDP → SP SAMLResponse (login): HTTP POST Binding only
- IDP → SP LogoutRequest (IDP-initiated SLO): HTTP POST Binding only — Redirect Binding fails (no DEFLATE inflation)
- IDP → SP LogoutResponse: HTTP POST Binding only — Redirect Binding fails

Root cause of the limitation: `SAMLReceiver.getLogoutRequest()` and `getLogoutResponse()` both do `Base64.decode → DocumentBuilder.parse` with no `java.util.zip.Inflater` call. HTTP Redirect Binding requires DEFLATE inflation before XML parsing. Applies to all versions from v17u8 through v18u13HF (bytecode is identical across versions — confirmed by decompilation)

---

## Configuration Files
The connector loads its configuration in two ways, depending on whether a GAM state is available

### When state IS available (normal operation)
`Propiedades.init(state)` → `loadPropsFromGAM(state)` → calls `GetAuthenticationParmsFromState` (GAM API procedure)

The GAM API returns a Java Properties-format string. Key properties:

- `SamlEndpointLocation` — IDP SSO URL (where to send AuthnRequest)
- `SingleLogoutLocation` — IDP SLO URL (where SP sends LogoutRequest)
- `AssertionConsumerServiceURL` — SP ACS URL. Typically `/saml/gam/signin`
- `NameIDPolicyFormat` — e.g. `urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress`
- `DisableSingleLogout` — `false` = SLO enabled
- `KeyCrypt` — encryption key for connector secrets

### When state is NOT available (fallback)
`Propiedades.init(null)` → `loadFileProperty("generalProps.properties")` — loaded from classpath via classloader

This file must be in `WEB-INF/classes/generalProps.properties`. If missing: silent failure (Properties object = null → NPE on first property read)

### Attribute mapping file
`attProps.properties` — loaded from classpath when `getAttProperty()` is called (during assertion parsing in login)

Must be in `WEB-INF/classes/attProps.properties`. Maps SAML assertion attributes to internal connector fields:

```properties
AttNOMBRECOMPLETO=<IDP attribute name for full name>
AttUID=<IDP attribute name for UID>
AttPRIMERNOMBRE=givenName
AttSEGUNDONOMBRE=
AttPRIMERAPELLIDO=sn
AttSEGUNDOAPELLIDO=
AttDOCUMENTO=
AttPAISDOCUMENTO=
AttTIPODOCUMENTO=
AttPRESENCIAL=
AttCERTIFICADO=
AttTRUE=
```

If `attProps.properties` is missing: `Properties.load(null)` throws NullPointerException during login:

```
java.lang.NullPointerException
	at java.util.Properties$LineReader.readLine(Properties.java:434)
	at artech.security.saml.Propiedades.loadFileProperty(Propiedades.java:347)
	at artech.security.saml.Propiedades.getAttNOMBRECOMPLETO(Propiedades.java:293)
	at artech.security.saml.SAMLReceiver.getDataFromAssertion(SAMLReceiver.java:331)
	at artech.security.saml.servlet.SSO.doPost(SSO.java:90)
```

Login may still succeed partially (the attribute value is simply null), but user attribute mapping is lost

---

## Log Signatures
The connector logs via SLF4J. All messages go to the application server stderr log (not stdout where GAMTraces appear)

### Enabling connector logs (critical prerequisite)
The WAR ships with two conflicting SLF4J bindings in `WEB-INF/lib/`:
- `slf4j-nop-1.7.7.jar` — NOP logger (discards everything)
- `slf4j-simple-1.7.10.jar` — writes to System.err

SLF4J picks the first binding found → if `slf4j-nop` wins, all connector logs are silenced (no errors, no debug output). This is the default state in some builds

Evidence in the application server stderr log:

```
SLF4J: Found binding in […slf4j-nop-1.7.7.jar…]
SLF4J: Found binding in […slf4j-simple-1.7.10.jar…]
SLF4J: Actual binding is of type [org.slf4j.helpers.NOPLoggerFactory]
```

Fix: delete `slf4j-nop-1.7.7.jar` from `WEB-INF/lib/` and restart Tomcat

Verification string (search in the application server stderr log after a logout attempt):

```
[LOGOUT.java: doPost] RelayState ==
```

If this appears → connector logs are active

### LOGOUT servlet strings
- `[LOGOUT.java: init]` — servlet initialization (Tomcat startup)
- `[LOGOUT.java: doPost] RelayState ==` — every request to `/saml/gam/signout`
- `[LOGOUT.java: doPost] alg =` — after reading SigAlg parameter
- `[LOGOUT.java: doPost] samlParameter == null` — SP-initiated path (Branch 1)
- `[LOGOUT.java: LogoutResponse]` — Branch 2 entered (LogoutResponse from IDP)
- `[LOGOUT.java: samlParameter]` — value of SAMLResponse parameter logged
- `[LOGOUT.java: doPost] LogoutRequest con firma valida` — IDP-initiated SLO, signature valid
- `[LOGOUT.java: doPost] NO Es firma valida` — IDP-initiated SLO, signature invalid (blank response, no redirect)
- `[LOGOUT.java: Logout] Logout Error` — LogoutResponse status is not Success / PartialLogout
- `[LOGOUT.java: Logout] Logout Error  ----  Response is null` — LogoutResponse object is null
- `[LOGOUT.java: doPost] catch GXCSAMLException e` — error in LogoutResponse parsing
- `[LOGOUT.java: doPost] catch DataFormatException e` — DEFLATE-related error
- `[LOGOUT.java: doPost] catch IllegalArgumentException e` — invalid argument during processing
- `[LOGOUT.java: doPost] catch SecurityException e` — security error
- `[LOGOUT.java: doPost] catch InvalidKeyException e` — signature key error
- `[LOGOUT.java: doPost] catch SignatureException e` — signature verification error
- `[LOGOUT.java: doPost] catch NoSuchAlgorithmException e` — unknown signature algorithm
- `[LOGOUT.java: logYError] catch IOException` — error redirecting to error page

### SAMLReceiver strings
Typo `SAMLReciver` is in the source, not here

- `[SAMLReciver.java: getLogoutRequest] catch SAXException` — XML parse failed on LogoutRequest (most common when IDP uses HTTP Redirect Binding — DEFLATE bytes reach the XML parser)
- `[SAMLReciver.java: getLogoutRequest] catch ParserConfigurationException` — XML parser config error
- `[SAMLReciver.java: getLogoutRequest] catch UnmarshallingException` — OpenSAML unmarshal error
- `[SAMLReciver.java: getLogoutRequest] catch IOException` — IO error reading LogoutRequest
- `[SAMLReciver.java: getLogoutResponse] catch SAXException` — XML parse failed on LogoutResponse
- `[SAMLReciver.java: getLogoutResponse] alg is empty` — SigAlg param not present. Reads alg from signature itself
- `[SAMLReciver.java: getStatusAssertion] catch SAXException` — XML parse error on SAMLResponse (login)

### Propiedades strings
- `[Propiedades.java: init] - state =` — `init()` called. Logs the state value
- `[Propiedades.java: init] - state state == null` — no state. Loads `generalProps.properties` from classpath
- `[Propiedades.java: init] - state != null` — state present. Loads from GAM API
- `[Propiedades - loadPropsFromGAM] start` — beginning GAM API call for config
- `[Propiedades - loadPropsFromGAM] result:` — config received from GAM API (followed by property values)
- `[Propiedades - loadFileProperty] configFile:` — file loaded successfully from classpath
- `[Propiedades.java: loadFileProperty] FileNotFoundException` — properties file not found in classpath
- `[Propiedades.java: loadFileProperty] IOException` — IO error reading properties file
- `[Propiedades.java: init] - updatePropsFromGAM =` — `updatePropsFromGAM()` called (IDP-initiated SLO path)

---

## Known Issues
### Issue 1: Blank screen on IDP-initiated SLO via HTTP Redirect Binding
Symptom: browser stuck at `/saml/gam/signout?SAMLRequest=lZ…` (GET with SAMLRequest param). No error in GAMTraces. No response from server (HTTP 200 with empty body)

Root cause: `SAMLReceiver.getLogoutRequest()` does `Base64.decode → DocumentBuilder.parse` with no DEFLATE inflation. HTTP Redirect Binding encodes the LogoutRequest as `Base64(DEFLATE(XML))`. After Base64-decode, the parser receives raw DEFLATE bytes (first byte `0x95` or similar, never `0x3C` = `<`). XML parser fails with:

```
[Fatal Error] :1:1: Content is not allowed in prolog.
```

Visible in the application server stderr log (not in GAMTraces). The exception is caught in LOGOUT.doPost and silenced by the SLF4J NOP logger. No HTTP response is written → blank screen

Confirmation string in logs (after enabling connector logs): `[SAMLReciver.java: getLogoutRequest] catch SAXException`

Affected versions: v17u8 through v18u13HF (bytecode identical across all — confirmed by decompilation)

Fix: configure the IDP to use HTTP POST Binding for the `SingleLogoutLocation` endpoint (`/saml/gam/signout`). With POST Binding, the LogoutRequest arrives as `Base64(XML)` in the POST body — no DEFLATE — and the connector parses it correctly

When does this appear: only when the IDP sends an IDP-initiated LogoutRequest back to the SP via HTTP Redirect Binding — typically as part of a "Global Logout" / "notify all SPs" behavior implemented by some IDPs. IDPs that send LogoutResponse directly (without a back-channel LogoutRequest to the originating SP), or that use HTTP POST Binding, do not trigger this issue

### Issue 2: NullPointerException on login — attProps.properties missing
Symptom: login eventually succeeds but user's full name or other attributes are null. Stack trace in the application server stderr log:

```
java.lang.NullPointerException
	at java.util.Properties$LineReader.readLine(Properties.java:434)
	at artech.security.saml.Propiedades.loadFileProperty(Propiedades.java:347)
	at artech.security.saml.Propiedades.getAttNOMBRECOMPLETO(Propiedades.java:293)
	at artech.security.saml.SAMLReceiver.getDataFromAssertion(SAMLReceiver.java:331)
	at artech.security.saml.servlet.SSO.doPost(SSO.java:90)
```

Root cause: `attProps.properties` is missing from `WEB-INF/classes/`. `ClassLoader.getResourceAsStream("attProps.properties")` returns null → `Properties.load(null)` → NPE

Fix: add `attProps.properties` to `WEB-INF/classes/` with the attribute name mappings for the IDP (see "Attribute mapping file" above)

### Issue 3: All connector logs are silenced (SLF4J NOP)
Symptom: issues above produce zero output in any log. "Traces not generating" even though `EnableTracing=1` is set

Clarification: `EnableTracing` controls GAMTraces (written to the application server stdout). Connector logs are a separate logging path via SLF4J → written to System.err (application server stderr). These are completely independent

Root cause: two conflicting SLF4J JARs in `WEB-INF/lib/` — `slf4j-nop` wins → all connector logs discarded

Fix: delete `slf4j-nop-1.7.7.jar` from `WEB-INF/lib/` and restart Tomcat. See "Enabling connector logs (critical prerequisite)" above for verification
