---
name: common-trace-analyzer
description: Universal trace analysis — Code-to-Log Correlation, SDT Parameter Correlation, 5-pass methodology
---

# GAM Trace Analyzer
- PREREQUISITE: Before using this skill, ensure traces are ACTIVATED
- See [GAM Debugging Core](./common-debugging.md) for how to:
	* Enable GAM tracing (boolean `0`/`1` switches — master + per-repository) via Backoffice or API
	* Configure log output (web.config, appsettings.json, log4j)
	* Capture stdout/stderr per generator (.NET Framework, .NET Core, Java)
	* Use Playwright for browser-based reproduction with screenshots
	* Run SQL-level diagnosis when traces are insufficient
- This skill assumes traces are already captured and focuses on READING them

FUNDAMENTAL PRINCIPLE — Applies to ALL GAM versions and ALL log formats:
Every GAM trace is emitted inside a conditional block in the GAM code. If a trace does NOT appear
in the log, it means the condition evaluated to `false`. GAM never emits "else" or "condition was false"
messages. Therefore, the absence of an expected trace is the strongest diagnostic evidence — stronger
than any error message. This principle is the foundation of all trace analysis in this skill, regardless of log format

- Dependencies: This skill integrates methodology from:
	* [GAM Debugging Core](./common-debugging.md) — How to ACTIVATE and capture traces
	* [GeneXus Patterns for GAM Integration](./common-genexus-patterns.md) — GeneXus constructs used by GAM
	* [GAM Behavioral Patterns](./common-behavioral-patterns.md) — Verified behavioral patterns
	* The Nexa skill references — GeneXus language source of truth
	* gx-debug-output methodology — For when GAM traces are insufficient
	* code-reviewer pass structure — Systematic multi-pass analysis

---

## Log Format Detection (CRITICAL — Do This First)
GAM uses two distinct trace formats depending on the version. ALWAYS detect the format before analysis

### Legacy `GAMTrace` format (up to v18 Upgrade 14)
```
GAMTrace-SomeModule SomeAction: somevalue
GAMTrace-Oauth20-Token Response:{"access_token":"abc123",…}
GAMTrace-&LoginTmpPar SET:{"RepositoryGUID":"e755…","SessionType":3}
```

How to identify: Lines contain `GAMTrace-` prefix

### Structured JSON format (v18 Upgrade 15+ / GeneXus Beta)
```
2026-04-09 10:39:24,380 [11] DEBUG genexus.security.api.GAMAuthenticationLogin - AuthenticationType-Impersonate - {"data":{"isImpersonate":false,"AuthenticationTypeName":"local"}}
2026-04-09 10:39:09,400 [11] DEBUG genexus.security.api.GAMExternalAuthenticationInputValidParam - Authentication-OAuth-20
```

Structure: `<timestamp> [<thread>] <level> <logger> - <event_label> - {"data":{<payload>}}`

Or without data: `<timestamp> [<thread>] <level> <logger> - <event_label>`

Components:
- `logger` — `genexus.security.api.<ProcedureName>`, e.g. `genexus.security.api.GAMAuthenticationLogin`
- `event_label` — Descriptive event (kebab-case or lifecycle), e.g. `Start_Method`, `End_Method`, `Start_Sub-ReadWebSession`, `SecurityGAMLocal-User-and-Password-OK`
- `{"data":{…}}` — Structured JSON payload via `GAMDictionary`, e.g. `{"data":{"isImpersonate":false,"AuthenticationTypeName":"local"}}`

Lifecycle labels (from `GAMLogMessageTypes` domain):
- `Start_Method` — Procedure entry point (parameters in `{"data":{…}}`)
- `End_Method` — Procedure exit point (results in `{"data":{…}}`)
- `Start_Sub-<SubName>` — Subroutine entry
- `End_Sub-<SubName>` — Subroutine exit

CRITICAL difference: Format B has an additional runtime condition: `Log.IsDebugEnabled()` must return `true`. This means `log.config` must have root level `DEBUG` or `ALL` — not just for file output, but also as a condition for GAM to emit the trace at all

### Quick Detection Command
```bash
# Detect format in a log file:
if grep -q "GAMTrace-" log.txt; then
		echo "FORMAT A (Legacy GAMTrace — up to v18u14)"
elif grep -q "DEBUG genexus.security.api\." log.txt; then
		echo "FORMAT B (Structured JSON — v18u15+)"
else
		echo "NO GAM TRACES FOUND — check activation (see common-debugging.md)"
fi
```

### Migration Notes
- In Beta/u15+, 5 legacy `GAMTrace-` lines remain active (3 SAML KeyStore traces + 2 misc). These coexist with the new format in the same log
- The structured payload serializes to `{"data":{…}}` — this is NOT a raw `SDT.ToJson()` anymore
- The logger name in the log shows `genexus.security.api.ProcName` (framework strips internal prefixes)

---

## Parsing v18u15+ Format B
This section is a parsing reference for Format B. Signatures for specific flows are catalogued in [GAM Trace Signatures Catalog](trace-signatures-catalog.md) Format B section. Canonical flow definition for OAuth 2.0 SIGNIN and SLO: [GAM-as-SP — OAuth 2.0 Common Flow (observable in trace)](../authentication/external-providers/oauth20/common/oauth20-common-flow.md)

### Line anatomy
```
<timestamp> [<thread>] DEBUG genexus.security.api.<ProcName> - <Phase> - {"data":{…}}
```

- `<timestamp>` — yyyy-MM-dd HH:mm:ss,fff
- `[<thread>]` — numeric thread id, the strongest within-log correlation key
- `genexus.security.api.<ProcName>` — cross-generator logger
- `<Phase>` — lifecycle label or named event, see vocabulary below
- `{"data":{…}}` — payload, single-level JSON object (see below)

### Parsing the payload
- The outer shape is always `{"data":{<body>}}`. `<body>` is a single-level JSON object. Nested objects (for example `{"data":{"User":{…}}}`) are values, not additional payload containers
- Claim names and SDT field names appear verbatim as JSON keys (for example `sub`, `email`, `SessionType`, `CacheConnectionCli`, `Errors`)
- Empty `""`, `[]`, `{}` are diagnostic clues — they propagate downstream
- `Start_Method` payloads typically carry `Parm1`, `Parm2`, … in entry order
- `End_Method` payloads typically carry `retval` and / or `Errors`

### Phase vocabulary
- `Start_Method` / `End_Method` — procedure entry / exit
- `Start_Sub-<SubName>` / `End_Sub-<SubName>` — named sub-routine entry / exit, marks internal branch taken
- `<NamedEvent>` — flow checkpoint specific to a procedure. Observed examples:
	* `GoToIP-AuthType-OAuth20` — about to redirect browser to external IDP
	* `ReturnFromIP-Add-Header-AuthType-OAuth20` — building Authorization header for token request
	* `ReturnFromIP-Add-GrantType-AuthType-OAuth20` — building `grant_type` body param
	* `ReturnFromIP-UserInfo-AuthType-OAuth20` — about to call UserInfo endpoint
	* `Cache-NotFound` / `Cache-Found` — cache miss / hit
	* `ValidWhenIsIDP-True` — IDP-side return handler matched
	* `Otherwise` — default branch of a nested When/Case

### Cross-generator scope
- `genexus.security.api.*` lines apply to all GeneXus generators (`.NET`, `.NET Framework`, `.NET Core`, Java). Cite these freely
- Lines under `GeneXus.*` (for example `GeneXus.Data.NTier.DataStoreProvider`, `GeneXus.Http.GxWebSession`, `GeneXus.Cache.InProcessCache`, `GeneXus.Metadata.ClassLoader`) are `.NET`-family-only stack frames. Do not rely on them when supporting Java deployments

### Following a flow through a Format B log
- Identify the thread id of the request of interest from the first suspect line — `[<N>]`
- Filter to that thread plus the cross-generator logger: grep by `[<N>]` and `genexus.security.api.`
- Read chronologically: `Start_Method` → intermediate `<NamedEvent>` and `Start_Sub-*` → `End_Method`
- `Start_Sub-*` lines mark which internal branch was taken. Absence of a `Start_Sub-*` that the domain flow would require = branch not taken
- Apply the 5-pass methodology below. Only the pattern library changes between Format A and Format B; the passes themselves are unchanged

---

## Input/Output Contract
### Input
- Log file(s) — Required — `.txt` or `.log` from the user
- Error description — Required — Symptom reported by the user
- Generator type — Preferred — .NET Framework / .NET Core / Java
- GAM version — Preferred — 18u14, 18u15+, Beta — determines log format (A or B, see above)
- Log format — Auto-detect — Format A (`GAMTrace-`) or Format B (`genexus.security.api.*` JSON) — auto-detected in Pass 1

### Output
Extended diagnostic format — canonical in `SKILL.md` § OUTPUT (Root Cause → Evidence → Code Path → Recommendation → Confidence). Do not restate here

---

## Foundational Principle 1: Code-to-Log Correlation
- CRITICAL: Every GAM trace that does NOT appear in the log means its containing condition evaluated to `false`. The absence of a trace IS the evidence. This applies to both Format A (Legacy) and Format B (v18u15+)

### Behavioral Rules
- GAM traces are emitted only when the tracing condition is true. Every trace lives inside a conditional block
- GAM never emits "else" or "condition was false" traces — it only emits when the condition is true. Silence IS the evidence
- Tracing requires `EnableTracing > 0` in the repository configuration
- v18u15+ additional condition: `Log.IsDebugEnabled()` must also be `true`, which depends on `log.config` root level being `DEBUG` or `ALL`

### EXAMPLE — Silent Skip (Correct vs Incorrect)
CORRECT (trace present = Authorization header added):
```
GAMTrace-Oauth20-Token Response:{"access_token":"abc123","token_type":"Bearer",…}
GAMTrace-GAMRemote AddHeader: Authorization:Bearer abc123        <- PRESENT
GAMTrace-Oauth20-User Response:{"email":"user@example.com",…}
```

INCORRECT (trace absent = Silent Skip Anti-Pattern):
```
GAMTrace-Oauth20-Token Response:{"access_token":"abc123","token_type":"",…}
																																							<- ABSENT: "AddHeader: Authorization"
Code 114: Token no valido                                                     <- Downstream error
```

The empty `token_type` causes GAM to skip the Authorization header — the condition for adding it was `false`

### EXAMPLE — Silent Skip (v18u15+)
CORRECT (complete lifecycle: Start -> validation OK -> End):
```
DEBUG genexus.security.api.GAMAuthenticationLoginGAMLocal - Start_Sub-SecurityGAMLocal-Local
DEBUG genexus.security.api.GAMAuthenticationLoginGAMLocal - SecurityGAMLocal-User-Local
DEBUG genexus.security.api.GAMValidUserRepositoryAccess - End_Method - {"data":{"Errors":[]}}
DEBUG genexus.security.api.GAMAuthenticationLoginGAMLocal - SecurityGAMLocal-User-and-Password-OK    <- PRESENT
```

INCORRECT (trace `SecurityGAMLocal-User-and-Password-OK` absent):
```
DEBUG genexus.security.api.GAMAuthenticationLoginGAMLocal - Start_Sub-SecurityGAMLocal-Local
DEBUG genexus.security.api.GAMAuthenticationLoginGAMLocal - SecurityGAMLocal-User-Local
DEBUG genexus.security.api.GAMValidUserRepositoryAccess - End_Method - {"data":{"Errors":[…]}}
																																																			<- ABSENT: "SecurityGAMLocal-User-and-Password-OK"
```

The non-empty `Errors` indicates that `GAMValidUserRepositoryAccess` failed, so the password check never executes

---

## Foundational Principle 2: SDT Parameter Correlation
- When a trace contains a JSON dump, it is a snapshot of the runtime state at that exact point
- Extract each field and use it to predict which branches will execute next
- This applies to both Format A (inline JSON) and Format B (`{"data":{…}}` payloads)

### How It Works
- Format A (Legacy): JSON dumps appear inline after the trace label, e.g.:
	`GAMTrace-&LoginTmpPar SET:{"RepositoryGUID":"e755…","ConnectionName":"",…"SessionType":3}`
- Format B (v18u15+): JSON dumps appear wrapped in `{"data":{…}}`, e.g.:
	`DEBUG genexus.security.api.RepositoryLogin - Login-GAMSessionJSON_SDT: - {"data":{"GAMSessionJSON_SDT":{"InitialProperties":[],"TokenSSORest":"SSORT!…","PKCE_challange":"7ZPd…","PKCE_method":"S256"}}}`
	`TokenSSORest` non-empty = SSO REST active — see [SSO REST — Single Sign-On for REST Services](../authentication/sso-rest/domain-sso-rest.md)

### CONSTRAINTS
- SDT JSON dumps are immutable snapshots of the state at that moment
- Empty fields (`""`) in the JSON are critical clues — they propagate empty downstream
- Comparing JSON dumps between procedures reveals fields lost during serialization

### CONSTRAINTS — v18u15+ (additional)
- In v18u15+ the JSON dumps also include empty `[]` and `{}` as critical clues
- The payload is always wrapped in `{"data":{…}}` — the actual content is one level deep
- `Start_Method` traces typically contain input parameters (`Parm1`, `Parm2`, etc.)
- `End_Method` traces typically contain results and `Errors`

### EXAMPLE — Field lost between procedures
```
// Procedure A output:
GAMTrace-&LoginTmpPar SET:{"RepositoryGUID":"e755…","ConnectionName":"","SessionType":3}
																											 ConnectionName empty HERE

// Downstream behavior: the empty ConnectionName propagates through the callback
// -> connection resolution fails -> Error Code 1
```

### EXAMPLE — Parameter tracking between procedures (v18u15+)
```
// Procedure entry with parameters:
DEBUG genexus.security.api.GAMGetConnectionGAM - Start_Method - {"data":{"Parm1":"","Parm2":"","Parm3":false}}
																																					Parm1 empty = ConnectionName did not arrive

// Procedure exit with result:
DEBUG genexus.security.api.GAMGetConnectionGAM - End_Method - {"data":{"CacheConnectionCli":{"Key":"6f42…","Name":"GAM_Dev_IDPu15…"},"Errors":[]}}
																																				CacheConnectionCli populated = connection OK

// If End_Method shows non-empty Errors -> the call failed
DEBUG genexus.security.api.GAMGetConnectionGAM - End_Method - {"data":{"CacheConnectionCli":{},"Errors":[{"Code":1,"Message":"Connection not found"}]}}
```

---

## Foundational Principle 3: First Error Priority
CRITICAL — Apply this before any other analysis step. The first error that appears chronologically in the log is almost always the root cause. Every subsequent error of the same Code is a cascade triggered by that first failure. Analyzing cascades produces wrong diagnoses

The client does NOT have access to GAM internal source code. The goal of this principle is not to read GAM code — it is to identify the root cause from observable log evidence and give the client actionable configuration or code changes to fix it

### Methodology
- Scan the full log for the FIRST occurrence of an error line:
	* Format A: lines containing `&Errors:[{"Code":` or `Error when <procedure> - Errors:[`
	* Format B: `End_Method` lines containing `"Errors":[{"Code":`
- Note the Error Code and Message from that first occurrence
- Count repetitions — if the same Code repeats, it is a cascade; work only with the first instance
- Identify which GAM procedure produced the first error:
	* Format A: read the `Caller:` stack trace that appears near the first error; the innermost `genexus.security.api.<ProcName>.privateExecute` is the failing procedure
	* Format B: the `genexus.security.api.<ProcName>` logger name on the failing `End_Method` line
- Cross-reference the Error Code with the Error Pattern Catalog in this skill and in the domain files
- Based on the match, determine the next step — there are three possible outcomes:
	* Known client-side fix: the error is documented and the client can configure or change something — provide the specific action
	* Possible GAM bug or unresolvable on the client side: the error is known but has no client-side fix — state this clearly and escalate
	* Unknown: the error or context is not in the catalog — do NOT invent a diagnosis; state that the cause could not be determined from available evidence and list what additional information (more traces, GAM version, generator) would help
- Only then apply Passes 1-5 to confirm the code path if needed

### Example — Cascade vs. Root Cause
A log with 443 lines repeating Error Code 30:

```
Line 18:  &Errors:[{"Code":30,"Message":"La conexión al GAM no fue encontrada..."}]
Line 45:  &Errors:[{"Code":30,"Message":"La conexión al GAM no fue encontrada..."}]
... repeats 20+ times
```

- Wrong diagnosis: GAM has a recurring connection problem
- Correct diagnosis (applying this principle):
	* First instance: line 18
	* Stack trace shows `getfileconnectionkey.privateExecute` looking for `./connection.gam`
	* `&SysConnCfgKey:` is empty — the file was not found in the process working directory
	* Error Code 30 means: connection configuration file not found
	* Root cause: `connection.gam` is not present where the CLI process runs
	* Cascade: every subsequent call needing a DB connection repeats Code 30
- Fix for the client: copy `connection.gam` to the JAR execution directory, or set the `GAM_CONNECTION_KEY` environment variable with the connection GUID

Fixing one thing eliminates all 443 error lines

### Additional rules
- When the first error appears before the user's code explicitly calls GAM (during logging or initialization) — the problem is environment/configuration, not business logic
- For CLI/daemon processes (`java -jar`), `httpContext.getDefaultPath()` returns empty; GAM falls back to `./` for `connection.gam` resolution — the file must be in the working directory or `GAM_CONNECTION_KEY` must be set
- For Format B: the first `End_Method` with `"Errors":[{…}]` is the starting point; `"Errors":[]` always means success
- After identifying the procedure name from the stack/logger, check if that procedure appears in the Error Pattern Catalog below — known fixes for common procedures are documented there
- If the error code or procedure is not in the catalog, do NOT invent a fix — the cause may be a GAM bug, an undocumented edge case, or something requiring internal investigation; say so explicitly and ask for more information or escalate

---

## Workflow — 5 Passes (adapted from code-reviewer methodology)
### Pass 1 — Segment and classify (2 minutes)
Objective: Understand the log structure and classify the domain

Step 0 — Detect format (v18u15+):
```bash
# Auto-detect format
if grep -q "GAMTrace-" log.txt; then
		FORMAT="A"  # Legacy (up to v18u14)
elif grep -q "DEBUG genexus\.security\.api\." log.txt; then
		FORMAT="B"  # Structured JSON (v18u15+)
else
		echo "NO GAM TRACES — check activation (common-debugging.md)"
		exit 1
fi
```

Step 1 — Segment by domain:

```bash
# Segment by domain (PowerShell/bash)
# Auth:
grep -i "GAMTrace-Oauth\|GAMTrace-SAML\|GAMTrace-Login\|GAMTrace-OIDC\|GAMTrace-GAMRemote" log.txt

# SLO:
grep -i "GAMTrace-SLO\|GAMTrace-Logout\|TokenToFinish\|LogoutType" log.txt

# Authz:
grep -i "GAMTrace-CheckPermission\|GAMTrace-Session\|GAMTrace-Application" log.txt

# 2FA:
grep -i "GAMTrace-2FA\|GAMTrace-OTP\|GAMTrace-TOTP\|AuthenticationType2FA" log.txt

# SAML:
grep -i "GAMTrace-SAML" log.txt
```

v18u15+ — Segment by domain (Structured JSON):
```bash
# Auth (OAuth 2.0 / OIDC):
grep -i "GAMExternalAuthentication\|GAMRemote\|Oauth20\|ValidApplicationOAuth\|ValidScopes\|GAMAuthenticationLogin" log.txt
# SLO:
grep -i "SLO\|Logout\|LogoutType\|TokenToFinish\|DeleteWebSession" log.txt
# Session:
grep -i "GAMSessionAPI\|GAMSessionAPIRead\|GAMGenerateToken\|WebSession\|AntiFixation" log.txt
# 2FA:
grep -i "2FA\|OTP\|TOTP\|TwoFactor\|SecondFactor" log.txt
# SAML:
grep -i "SAML\|saml" log.txt
# Connection/Config:
grep -i "GAMGetConnectionGAM\|GAMGetCache\|GAMSetCache\|GAMLoadRepository\|connection\.gam" log.txt
# Extract only GAM traces (filter framework noise):
grep "DEBUG genexus\.security\.api\." log.txt
```

Classify by state prefix (if state parameters appear in traces):
- `GRESTD` — Standard auth — [GAM Authentication](../authentication/external-providers/common/domain-auth.md)
- `SLOInt` — Single Log Out — [GAM Single Log Out (SLO)](../logout/domain-slo.md)
- `OA2STD` — OAuth 2.0 — [GAM Authentication](../authentication/external-providers/common/domain-auth.md)
- No prefix — Local login or CRUD — [GAM Authentication](../authentication/external-providers/common/domain-auth.md) or [GAM Authorization — Permission Evaluation](../authorization/domain-authz.md)

Format B — Additional classification by logger name:
- `GAMExternalAuthentication*`, `GAMRemote*` — Auth (OAuth callback) — [GAM Authentication](../authentication/external-providers/common/domain-auth.md)
- `GAMAuthenticationLogin*` — Login (local/remote) — [GAM Authentication](../authentication/external-providers/common/domain-auth.md)
- `GAMSessionAPI*` — Session management — [GAM Session and Token Reference](../authentication/external-providers/common/domain-session-token.md)
- `RepositoryInput*`, `RepositoryGet*` — IDP flow — [GAM Authentication](../authentication/external-providers/common/domain-auth.md)
- `GAMGetConnection*`, `GAMGetCache*` — Config/Connection — [GAM Connection & Configuration Encyclopedia](../multi-tenant/domain-connection-config.md)
- `ValidApplication*` — App validation — [GAM Authorization — Permission Evaluation](../authorization/domain-authz.md)
- `GAMDeleteWebSession*` — SLO/Logout — [GAM Single Log Out (SLO)](../logout/domain-slo.md)

### Pass 2 — Code-to-Log Correlation (5 minutes)
Objective: Build checklist of present/absent traces

- Identify the expected trace sequence for the domain (see "Search patterns by domain" below)
- Search for each expected trace in the log
- Construct checklist:

```
GAMTrace-==+++GAMAuthenticationLogin ====== START ---
GAMTrace-GAMAuthenticationLogin - &User:admin
GAMTrace-GAMAuthenticationLogin 2FA - Valid &AuthenticationType2FA:0
GAMTrace-GAMAuthenticationLogin - Session created       <- ABSENT
```

v18u15+ — Construct checklist with JSON format:
```
genexus.security.api.GAMAuthenticationLogin - Start_Method - {"data":{…}}
genexus.security.api.GAMAuthenticationLogin - AuthenticationType-Impersonate - {"data":{"isImpersonate":false}}
genexus.security.api.GAMAuthenticationLoginGAMLocal - SecurityGAMLocal-User-and-Password-OK
genexus.security.api.GAMUserValidRequiredDataRepository - End_Method - {"data":{"CompleteUserDataOK":true}}
genexus.security.api.GAMSessionAPI - Start_Sub-LoginUserInWebSession       <- ABSENT
```

- For each absent trace, determine what condition was false — the absence means the branch was not taken

v18u15+ — Tips for building the checklist:
- Search the procedure name in the logger: `grep "genexus.security.api.GAMAuthenticationLogin " log.txt`
- `Start_Method` / `End_Method` mark the procedure boundaries
- An `End_Method` with empty `"Errors":[]` = success; with `"Errors":[…]` = failure
- Absence of `End_Method` after `Start_Method` = the procedure aborted or threw an exception

### Pass 3 — SDT Parameter Correlation (5 minutes)
Objective: Track SDT values across procedure boundaries

- Find all JSON dumps in the log (Format A: `GAMTrace-…:{`, Format B: `{"data":{`)
- Parse each JSON — extract fields and values
- Compare output of Procedure A with input of Procedure B
- Mark fields that are empty or changed between procedures

v18u15+ — Additional steps for JSON format:
- Find all `{"data":{…}}` payloads: `grep '"data":{' log.txt`
- Parse the JSON inside `"data"` — extract fields and values
- Use `Start_Method`/`End_Method` as clear boundaries between procedures
- Then compare and mark empty fields same as in the legacy format

Critical SDTs:
- `APIStateClientPar` — Look for `FromURL`, `ConnectionName`, `GAMLogoutType` — [GAM Session and Token Reference](../authentication/external-providers/common/domain-session-token.md)
- `GAMExternalAuthenticationInputSDT` — Look for `OriginType`, `OriginTypeValue` — [GAM External Authentication Entry Point](../authentication/external-providers/common/domain-extauthinput.md)
- `GAMExternalAuthenticationOutputSDT` — Look for `token_type`, `access_token` — [GAM Authentication](../authentication/external-providers/common/domain-auth.md)
- `LoginTmpPar` — Look for `RepositoryGUID`, `SessionType` — [GAM Session and Token Reference](../authentication/external-providers/common/domain-session-token.md)

v18u15+ — Additional SDTs/Fields in JSON format:
- `CacheConnectionCli` — Look for `Key`, `Name`, `Repository`, `Type` — [GAM Connection & Configuration Encyclopedia](../multi-tenant/domain-connection-config.md)
- `GAMSessionJSON_SDT` — Look for `TokenSSORest`, `PKCE_challange`, `PKCE_method` — [GAM Session and Token Reference](../authentication/external-providers/common/domain-session-token.md); `TokenSSORest` non-empty = SSO REST active — see [SSO REST — Single Sign-On for REST Services](../authentication/sso-rest/domain-sso-rest.md)
- `GAMApplicationValidScopes_SDT` — Look for `ValidatedScopes`, `gam_user_data`, `openid` — [GAM Authentication](../authentication/external-providers/common/domain-auth.md)
- `Errors` array — `[]` = OK, `[{Code, Message}]` = failure — Cross-cutting
- `GAMExternalAuthenticatinInputSDT` — Same as `GAMExternalAuthenticationInputSDT` (known typo in trace output) — [GAM External Authentication Entry Point](../authentication/external-providers/common/domain-extauthinput.md)

v18u15+ — Start/End pattern for tracking a complete procedure:
```
DEBUG genexus.security.api.ProcName - Start_Method - {"data":{"Parm1":"input1","Parm2":"input2"}}
… (intermediate traces) …
DEBUG genexus.security.api.ProcName - End_Method - {"data":{"Result":"output","Errors":[]}}
```

### Pass 4 — Cross-log Correlation (when multiple logs exist)
Objective: Correlate events between client-side and IDP-side

When the diagnosis involves redirect flows (OAuth, SAML, SLO):

- Order logs by timestamp (the redirect causes a time gap)
- Search for the state parameter as a thread linking logs
- Verify the state matches:
	* Client-side: `GAMTrace-State saved: <state>` -> redirect
	* IDP-side: `GAMTrace-ValidStateInDB OK State=<state>` -> callback
- If the state does not match or does not appear: State expired, CSRF, or `GAMLoginTmpDemon` cleaned it

### Pass 5 — Deep dive with gx-debug-output (if Passes 1-4 do not resolve)
Objective: When GAM traces are insufficient, insert debug statements in the generated code

- Read the gx-debug-output skill methodology

- Identify the gap: Where are the last present trace and the first absent trace?
- Locate in generated code (C# or Java depending on generator):
	* .NET: Look in `bin/` or deploy folder for the relevant assembly
	* Java: Look in `WEB-INF/classes/` for the generated class
- Insert strategic debug at decision points between the two traces:

	C# (.NET):
	```csharp
	Console.Error.WriteLine($"[DEBUG] GAM:{location} - {variable}={value}");
	```

	Java:
	```java
	System.err.println("[DEBUG] GAM:" + location + " - " + variable + "=" + value);
	```

- Reproduce the issue and capture `[DEBUG]` lines
- Analyze variable values to identify root cause
- Clean up debug statements

- CRITICAL: If the bug is in GeneXus generated code (not custom), the fix is in the GeneXus model (Transaction, Procedure, etc.), not in C#/Java. Refer to Nexa for model understanding

---

## Search patterns and error catalog
Per-domain expected trace sequences (Auth, SLO, SAML, 2FA/OTP) and the Error Pattern Catalog: [Trace Search Patterns and Error Catalog](trace-search-patterns.md) — load during Pass 2-4 of the workflow below

---

## Integration with Other Skills and Agents
### How to use this skill from an agent
```
1. Agent receives a log from the user
2. Read [common-trace-analyzer.md](./common-trace-analyzer.md) (this file)
3. Apply Foundational Principle 3 — First Error Priority:
   a. Find the FIRST error line in the log (Format A: &Errors:[{"Code": / Format B: "Errors":[{)
   b. Note the Error Code and Message
   c. Count repetitions to confirm cascade
   d. Identify the failing procedure from the stack trace (Format A) or logger name (Format B)
   e. Cross-reference with the Error Pattern Catalog
   f. If the root cause is clear -> go directly to Recommendation (skip to step 8)
4. Execute Pass 1 -> DETECT FORMAT (A or B) -> determine domain
5. Read the corresponding domain file
6. Read [common-genexus-patterns.md](./common-genexus-patterns.md) if needed to understand GeneXus code
7. Execute Passes 2-4 -> build evidence checklist (use patterns for detected format)
   If Passes 2-4 do not resolve -> execute Pass 5 (gx-debug-output methodology)
8. Produce output in the standard format (Root Cause + Evidence + Recommendation)
```

CRITICAL: Step 3 (First Error Priority) always runs first. Many cases are resolved in step 3 without needing the full 5 passes. The agent MUST detect the format in Pass 1 before applying any search pattern — Format A patterns on a Format B log produce zero matches

### When to consult other references
- The issue involves .NET vs Java differences — [GAM Cross-Generator Issues Encyclopedia](../environments/domain-cross-generator.md)
- The issue involves multi-tenant / connection.gam — [GAM Multi-Tenant & Repository Encyclopedia](../multi-tenant/domain-multitenant.md)
- The fix requires changes to the GeneXus model — The Nexa skill
- The issue requires debugging of generated code — The gx-debug-output skill
