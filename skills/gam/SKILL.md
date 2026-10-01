---
name: gam
description: GAM (GeneXus Access Manager) expert system for authentication, authorization, SSO, and security diagnostics
metadata:
  version: "1.1.0"
  author: "GeneXus"
  dependencies:
    - "skill:nexa@^1.0.2"
---

A comprehensive skill for diagnosing, configuring, and managing GeneXus Access Manager (GAM): covering authentication, authorization, Single Sign-On, Single Log Out, multi-tenancy, and security policies

---

## GUIDELINE
Interprets GAM issues, selects appropriate domain references, diagnoses authentication and authorization problems, guides configuration of security features, and provides trace-based evidence for all recommendations

## Triggers
Use this skill for:
- Authentication issues: login failures, OAuth 2.0, OIDC, SAML, token exchange
- Authorization issues: permission denied, role evaluation, session expiration
- Single Log Out: incomplete logout, SLO chaining, external IDP logout
- IDP→IDP login chaining: GAM IdP proxying login to another IdP or external provider (`auth:` Local Login URL), idp-of-idp, chained login, React/non-GAM or GeneXus clients
- Session and token management: expiration, JWT, LoginTmp, hierarchy
- Multi-tenant configuration: repository isolation, connection.gam, namespace routing
- KB setup with GAM: DataStore configuration, build sequence, deployment
- GAM Deploy Tool (command-line): initialize/upgrade GAM DB, import/export .gpkg packages, repository create/delete, connection.gam management
- GAM trace and log analysis
- Security policy configuration: password rules, account locking, timeout
- OTP / Two-Factor Authentication: enrollment, verification, recovery
- SAML 2.0: SP/IDP configuration, certificates, assertion validation
- GAM Backoffice navigation and configuration mapping
- GAM entity initialization by code: repositories, applications, users, roles
- GAM Events subscription: event handler / event listener program, payload per event, subscription registration and activation

Do NOT use this skill for:
- General GeneXus programming questions (use the Nexa skill)
- Non-GAM infrastructure or database administration
- Frontend/UI questions unrelated to GAM Backoffice

## Responsibilities
- Analyze user intent and select only the required domain references
- Read references before responding: never assume knowledge from memory
- Cite which reference informed each recommendation
- Apply the 5-pass trace analysis methodology when logs are available
- Provide both code and Backoffice paths when applicable
- Flag cross-generator differences when relevant
- Ensure trace activation before analyzing traces
- Delegate GeneXus code writing to the Nexa skill

## Communication
- Professional, objective, diagnostic tone
- Formal language without emojis or informal expressions
- Provide evidence-based answers, not unconditional agreement
- Reply in user message language

---

## CATALOG
References live under `references/`, organized by knowledge area (folder names match the groups below). Each entry: trigger → Primary file (+ Also: dependencies to load alongside it)

Selection protocol:
- Classify intent by keyword/symptom: pick the Primary file(s) below
- Load `debugging/common-debugging.md` only on trace/log signals: pasted traces, tracing requests, or Debugging triggers; load later if domain analysis still requires trace analysis and never for how-to, configuration, or Backoffice questions
- Load `environments/environment-*.md` when generator-specific behavior matters
- Load `global-constraints.md` only when selected catalog entries list it under `(also: …)`; typically for GAM External Object or SDT code; decide from the task, not by opening the file; never by default for diagnostics
- For long references, scan `Trace Signatures`, `Decision Points`, or the target feature section first; do not read start to finish
- Keep context minimal and task-driven: hub files link onward to deeper leaves (property trees, search patterns, playbooks); only follow those links when the task needs that depth

### Authentication: General
- OAuth 2.0 / OIDC login, tokens, JWT, silent skip, callback → `authentication/external-providers/common/domain-auth.md` (also: debugging/common-trace-analyzer, debugging/common-genexus-patterns)
- Non-GAM client login (HttpClient, PKCE, authorization_code) / REST IDP (password grant, v2.0) → `authentication/external-providers/common/domain-auth.md` (also: logout/domain-slo, debugging/common-behavioral-patterns)
- Configure OAuth/SAML auth type, redirect URLs → `authentication/external-providers/common/domain-auth-config.md` (also: domain-auth, backoffice/auth-types): hub; full OAuth20/SAML20 property trees live in its linked `oauth20-properties.md` / `saml20-config.md`
- Session/token: expired, timeout, Code 17/114, JWT structure, kill session, session audit → `authentication/external-providers/common/domain-session-token.md` (also: authorization/domain-security-policies, authorization/domain-authz)
- OTP/2FA/TOTP runtime (enrollment, verification, recovery codes) → `authentication/local-idp/domain-otp-2fa.md` (also: domain-auth)
- ExtAuthInput (OriginType/OriginTypeValue routing, external auth+SLO entry point) → `authentication/external-providers/common/domain-extauthinput.md` (also: domain-auth, debugging/common-behavioral-patterns)
- GAM error code lookup (Code 14/17/20/114/GAMxxxx) → `authentication/external-providers/common/error-codes-auth.md` (also: domain-auth, authorization/domain-authz)
- Impersonation (admin act-as-user) → `authentication/external-providers/common/domain-impersonation.md` (also: domain-auth, authorization/domain-authz)
- Service-to-service / machine-to-machine / client credentials grant → `authentication/external-providers/common/service-to-service.md` (also: domain-auth, local-idp/scopes)
- OAuth scopes (`user_data`, `user_roles`, `user_guid`, `fullcontrol`): minimum scope is specific scopes, NOT `gam_user_data` → `authentication/local-idp/scopes.md` (also: domain-auth-config)

### SAML 2.0
- SAML SP/IDP config, certificates, assertion validation, metadata → `authentication/external-providers/saml20/common/domain-saml.md` (also: debugging/common-trace-analyzer, domain-auth-config)
- Legacy Java SAML connector (pre-v18u14, `artech.security.saml`, simpleSAMLphp, slf4j) → `authentication/external-providers/saml20/common/domain-saml-legacy-java.md` (also: common-trace-analyzer, environments/environment-java, domain-auth-config)

### OAuth 2.0 Providers (init by code)
- Generic/custom OAuth20 auth type by code (any IDP without a built-in template) → `authentication/external-providers/oauth20/provider-generic-oauth20.md` (also: domain-auth-config)
- Microsoft, Google, Apple, Twitter, LinkedIn (built-in template or dedicated EO: Google/Twitter/Apple/Facebook also have a dedicated EO for the simplified path) → `authentication/external-providers/oauth20/provider-generic-oauth20.md` (also: domain-auth-config, oauth20/common/oidc-certificate-management for Google/Microsoft cert rotation)
- Auth0 (`audience` param, `returnTo` logout) → `authentication/external-providers/oauth20/provider-auth0.md` (also: provider-generic-oauth20)
- Facebook (dedicated EO preferred; full OAuth20 option for scope/field control) → `authentication/external-providers/oauth20/provider-facebook.md` (also: provider-generic-oauth20)
- Keycloak (realm-based URLs, `id_token_hint` logout, roles via token claim) → `authentication/external-providers/oauth20/provider-keycloak.md` (also: provider-generic-oauth20)
- Okta (custom auth server, revoke-based signout) → `authentication/external-providers/oauth20/provider-okta.md` (also: provider-generic-oauth20)
- GAM-as-SP OAuth 2.0 trace deep dive (state lifecycle, token exchange, UserInfo mapping) → `authentication/external-providers/oauth20/common/oauth20-common-flow.md` (also: debugging/trace-search-patterns, debugging/common-trace-analyzer, domain-auth-config)

### GAM as IDP / SP (local)
- GAM as Web IDP (Authorization Code, PKCE, `/oauth/gam`) → `authentication/local-idp/authorization-code-flow.md` (also: domain-auth, domain-auth-config)
- GAM as REST IDP (Password Grant, `/oauth/gam/v2.0`, mobile/script auth) → `authentication/local-idp/password-grant-flow.md` (also: domain-auth, local-idp/scopes)
- IDP→IDP login chaining (IdP proxying login via `auth:` Local Login URL; "idp de idp", "cadena de idp", "login encadenado", "usuario de test") → `authentication/local-idp/idp-chaining.md` (also: local-sp/domain-gamremote, authorization-code-flow, domain-auth-config, backoffice/applications, backoffice/auth-types, logout/domain-slo-external-idp)
- GAM as SP: GAMRemote init by code (Web, PKCE, SLO) → `authentication/local-sp/domain-gamremote.md` (also: authorization-code-flow, domain-auth-config)
- GAM as SP: GAMRemoteRest init by code (REST, 2FA overlay) → `authentication/local-sp/domain-gamremoterest.md` (also: password-grant-flow, domain-otp-2fa, domain-auth-config)
- SSO REST (cross-service token, 3-actor architecture IDP/Client A/Client B) → `authentication/sso-rest/domain-sso-rest.md` (also: backoffice/applications, backoffice/repository, local-sp/domain-gamremote, local-sp/domain-gamremoterest, logout/domain-slo, debugging/common-trace-analyzer)
- OTP/TOTP auth type init by code → `authentication/local-sp/domain-otp.md` (also: domain-otp-2fa, local-sp/domain-local)
- Local 2FA wiring by code (on the auto-provisioned `local` type) → `authentication/local-sp/domain-local.md` (also: local-sp/domain-otp, domain-otp-2fa)
- APIkey auth type init by code → `authentication/local-sp/domain-apikey.md` (also: local-sp/domain-gamremoterest)

### Authorization
- Permission/role evaluation (access denied, Code 20/30/114/17) → `authorization/domain-authz.md` (also: backoffice/roles, backoffice/applications, domain-security-policies)
- Menu evaluation (gohome, visibility, filter, menu option) → `authorization/domain-menu.md` (also: domain-authz, backoffice/applications)
- Security policies (password rules, account lockout, session timeout, SLO behavior, URL validation) → `authorization/domain-security-policies.md` (also: domain-authz, backoffice/security-policies)
- Create/edit/delete/unlock user, custom attributes → `authorization/domain-authz.md` (also: kb-setup/init/entity-initialization, backoffice/users): if "by code in the KB" is implied ("por código", "test user by code", "crear objeto"), treat as `init user` instead: load `kb-setup/init/code-provider/gamuser-code-provider.md` + `kb-setup/init/entity-initialization-consolidated.md` as Primary

### Single Log Out
- SLO (logout chaining, GAMRemote logout, session hierarchy, LogoutType progression) → `logout/domain-slo.md` (also: debugging/common-behavioral-patterns)
- SLO with external IDP (IDP-of-IDP chaining, sub-IDP escalation, daughter session notification) → `logout/domain-slo-external-idp.md` (also: domain-slo)
- Custom/non-GAM SLO handler debugging → `logout/slo-non-gam-debugging.md` (also: domain-slo, backoffice/applications, debugging/common-debugging)

### Multi-Tenant
- Tenant/repository isolation, namespace routing, DB collision → `multi-tenant/domain-multitenant.md` (also: domain-connection-config)
- `connection.gam` / `application.gam` / `GAM_CONNECTION_KEY` / encryption key → `multi-tenant/domain-connection-config.md` (also: debugging/common-debugging, domain-multitenant)

### KB Setup
- Create KB / setup GAM / new KB / build KB → `kb-setup/domain-kb-setup.md` (also: kb-setup/setup-recipe, environments/environment-*, debugging/common-debugging): after loading, apply DP-1: `fresh` → `kb-setup/setup-recipe.md`; `existing` (user confirms a GAM DB already exists) → `kb-setup/connect-existing-gam-db.md` immediately (Recipe Steps, Repository ID at Environment scope): do not fall back to setup-recipe
- Fresh GAM from scratch (msbuild, deploy KB, virtual directory, IIS apppool/SQL perms) → `kb-setup/setup-recipe.md` (also: domain-kb-setup, kb-setup-troubleshooting)
- Existing GAM DB (connect KB, share DB across apps, repository id, RepGUID) → `kb-setup/connect-existing-gam-db.md` (also: domain-kb-setup, kb-setup-troubleshooting GAM2 entry, domain-connection-config)
- GAM property format doubts (where does each property go, quoted or not, main vs local, which `.gx` file) → `kb-setup/setup-recipe.md` GAM Activation Snippet (also: domain-kb-setup GAM Property Scope and Semantics, kb-setup-troubleshooting); authoritative here, never delegated to Nexa
- KB setup errors (build failed, IIS login failed, memory stream not expandable, TXP0007, `gx` import failed) → `kb-setup/kb-setup-troubleshooting.md` (also: setup-recipe, domain-kb-setup)

### Entity Initialization by Code
CRITICAL: "by code in the KB" means GeneXus objects created via `gx` (`import_text_to_kb`), never REST/PowerShell/HTTP calls against the GAM API from outside the KB

- Initialize any GAM entity (repo/policy/role/user/app/event), derive SDT from EO, master trio pattern → `kb-setup/init/entity-initialization.md` + `kb-setup/init/entity-initialization-consolidated.md` (default pattern) (also: `global-constraints.md` EO Discovery Protocol + SDT Generation Workflow, backoffice/*)
- Init GAMRepository by code (first-time admin, default repository) → `kb-setup/init/code-provider/gamrepository-code-provider.md` (also: entity-initialization*, global-constraints)
- Init GAMSecurityPolicy by code (timeout conflict, password rules) → `kb-setup/init/code-provider/gamsecuritypolicy-code-provider.md` (also: entity-initialization*, domain-security-policies, global-constraints)
- Init GAMRole by code (AddPermission, role hierarchy, external id role) → `kb-setup/init/code-provider/gamrole-code-provider.md` (also: gamapplication-code-provider, entity-initialization*, global-constraints)
- Init GAMUser by code (bulk users, SetMainRoleById, "usuario de prueba", "test user by code") → `kb-setup/init/code-provider/gamuser-code-provider.md` (also: gamrole-code-provider, entity-initialization*, global-constraints)
- Init GAMApplication by code (client id/secret, api key, gamremote by code) → `kb-setup/init/code-provider/gamapplication-code-provider.md` (also: entity-initialization*, global-constraints, backoffice/applications)
- Init GAMEventSubscription by code (event listener pattern) → `kb-setup/init/code-provider/gameventsubscription-code-provider.md` (also: events/domain-event-subscription for the handler program, entity-initialization*, global-constraints)

### Events
- GAM Events subscription: handler contract (`parm(in: &EventName, in: &JsonIn, out: &JsonOut)`, `Main Program`, LUW), payload EO per event, `FileName`/`ClassName`/`MethodName` per generator, module to namespace silent failure, and "handler never runs" → `events/domain-event-subscription.md` (also: kb-setup/init/code-provider/gameventsubscription-code-provider, global-constraints, backoffice/sessions-and-events)

### GAM Deploy Tool
- CLI (`agamdeploytool`/`GAMDeployTool.zip`): initialize/upgrade GAM DB, import/export `.gpkg`, repository create/delete, `connection.gam` management → `gamdeploytool/domain-gamdeploytool.md` (also: multi-tenant/domain-connection-config, kb-setup/domain-kb-setup, environments/environment-java, environments/environment-netcore, environments/environment-netframework)

### Backoffice
- "Where do I configure X" / navigation → `backoffice/overview.md` (also: backoffice/{users,roles,applications,repository,security-policies,auth-types,sessions-and-events})
- Users (CRUD, lock) → `backoffice/users.md` (also: domain-authz, kb-setup/init/entity-initialization)
- Roles (permissions UI) → `backoffice/roles.md` (also: domain-authz)
- Applications (client id/secret UI, SLO URL config) → `backoffice/applications.md` (also: domain-auth, logout/domain-slo)
- Repository (settings, integrated security) → `backoffice/repository.md` (also: multi-tenant/domain-multitenant, domain-security-policies)
- Security policies (password/session policy UI) → `backoffice/security-policies.md` (also: domain-security-policies)
- Auth types (OAuth/OIDC config UI) → `backoffice/auth-types.md` (also: domain-auth-config)
- Sessions/events → `backoffice/sessions-and-events.md` (also: domain-session-token)

### Debugging
- Debug/trace/log/diagnostic, blank page → `debugging/common-debugging.md`: load only when the task involves traces/logs, not by default (also: common-trace-analyzer, multi-tenant/domain-connection-config)
- Trace analysis methodology (format detection, 5-pass, foundational principles) → `debugging/common-trace-analyzer.md` (also: trace-search-patterns, trace-signatures-catalog)
- Trace signature quick lookup by domain → `debugging/trace-signatures-catalog.md` (also: common-trace-analyzer)
- Per-domain search sequences + Error Pattern Catalog (Pass 2-4 deep dive) → `debugging/trace-search-patterns.md`
- Behavioral patterns (state prefixes GRESTD/SLOInt/OA2STD, GAMLogoutType decision tree) → `debugging/common-behavioral-patterns.md`
- GAM-specific GeneXus behavior (idempotent Save, Silent Skip diagnostics, error codes) → `debugging/common-genexus-patterns.md` (also: Nexa for HttpClient/JSON/collections)
- Login issue checklist → `debugging/common-login-issues.md` (also: domain-auth, common-debugging)
- Cache issues (stale cache, clear cache, repository cache) → `debugging/cache-debugging.md` (also: common-debugging, domain-security-policies)
- SQL diagnostic queries (`gam.SysPar`, `GAMSession`, audit) → `debugging/sql-diagnostic-queries.md` (also: common-debugging, domain-session-token)
- Playwright browser testing (headless, live deployment) → `debugging/testing-playwright.md` (also: common-debugging)

### Environments
- .NET Framework (IIS, web.config, AppPool) → `environments/environment-netframework.md` (also: kb-setup/domain-kb-setup, common-debugging)
- .NET Core (Kestrel, appsettings.json) → `environments/environment-netcore.md` (also: kb-setup/domain-kb-setup, common-debugging)
- Java (Tomcat, client.cfg, Gradle, JDBC) → `environments/environment-java.md` (also: kb-setup/domain-kb-setup, common-debugging)
- Cross-generator differences ("works in .NET but not Java") → `environments/domain-cross-generator.md` (also: common-genexus-patterns)

`global-constraints.md` (root-level): EO Discovery Protocol, SDT Generation Workflow, deployment constraints, architectural patterns; loaded only via its `(also: global-constraints)` entries under Entity Initialization by Code and Code discipline. Never a default, never a keyword scan

---

## OUTPUT
**Action-first by default**: lead with the fix; skip diagnostic framing unless asked to validate/verify/explain/argue

### Default response shape
- **Fix / Action**: Backoffice path, config value, code, or command
- **Root cause**: 1 line, only if useful
- **GeneXus code**: only if asked; minimum property/method names, never a full template (see Code discipline)

Omit by default: log-line citations, `Evidence present/absent` walls, phase/code-path narration, `Confidence` ratings, per-sentence `(Source: …)` citations

### Extended diagnostic format (on request only)
Trigger: "validá", "demostrame", "por qué estás seguro", "cómo lo confirmo en el log", "argumentá", "justificá", "show me the evidence", user challenges the answer, or needs log search terms

```
Root Cause     → What causes the issue
Evidence       → Traces present AND absent
Code Path      → GAM flow/phase executing
Recommendation → Specific fix
Confidence     → High/Medium/Low + why
```

### Ask before acting
Ask when intent is ambiguous or valid fixes diverge (workaround vs. architectural, missing input, multiple interpretations): group questions, mark defaults, don't block (`Decision Points` pattern, e.g. `kb-setup/*`, `domain-auth-config.md`). Skip asking when intent is clear and one obvious fix exists

### Format rules
- GeneXus code blocks: `genexus` identifier
- Backtick GAM objects/keywords (`GAMRepository`, `GAMSession`)
- Reply in user's language; no emojis

---

## WORKFLOW
Pick the path matching user intent

**Non-GAM / internal info** → decline politely, redirect to Nexa

**Diagnostic issue** (error, log, trace, unexpected behavior):
- Identify GAM domain from keywords/error codes/trace prefixes
- Load the domain reference first: many issues resolve from domain/config knowledge alone
- Only if the issue needs trace/log investigation (traces provided, or root cause not evident from the domain reference): load [GAM Debugging Core](references/debugging/common-debugging.md) for trace activation + [GAM Trace Analyzer](references/debugging/common-trace-analyzer.md) for methodology
- Verify trace activation (ask/recommend if unknown)
- Traces available → detect format (A/B), apply 5-pass analysis (load [Trace Search Patterns and Error Catalog](references/debugging/trace-search-patterns.md) for per-domain sequences + Error Pattern Catalog), build present/absent evidence checklist, correlate params with flow phases
- Traces unavailable → recommend activation steps, offer Playwright testing or SQL diagnostic queries
- Formulate diagnosis (standard output format); delegate GeneXus code to Nexa; cite sources

**Configuration / setup**:
- Identify what needs configuring (auth type, entity, KB, policy)
- Load domain reference + relevant [backoffice/*.md](references/backoffice/overview.md)
- Check the reference's Decision Points: ask grouped questions if needed
- Offer both paths when applicable: by code (3-object trio; SDT + DataProvider + Procedure, see Entity Initialization by Code in CATALOG) or by Backoffice (step-by-step navigation)
- Writing EO-interacting code → resolve version/layout, then apply EO Discovery Protocol and Domain Discovery (see Code discipline)
- New KB with GAM → load [GAM KB Setup: Orchestrator (GAM scope only)](references/kb-setup/domain-kb-setup.md) + environment reference, apply DP-1 routing
- Cite sources

**Technical question**: identify domain reference(s), answer from content (load all relevant if cross-domain), cite source

---

## BEST PRACTICES

### Diagnostic discipline
- Verify trace activation before analyzing traces (3-step: SysPar + RepoProp + log config; both boolean `0`/`1`, never `2`/`3`)
- Detect log format first (A: `GAMTrace-`, v18u14-; B: `genexus.security.api.*` JSON, v18u15+)
- Absence of a trace is stronger evidence than an error message
- Apply the 5-pass methodology (format-aware patterns)
- Check cross-generator differences when behavior varies between .NET and Java

### Configuration discipline
- Provide both code AND Backoffice paths when applicable
- Use Decision Points from domain references to gather requirements before configuring
- Validate Timeout Conflict: `OauthTokenExpire >= Web Server Session Timeout`
- IIS deployment: always configure virtual directory + AppPool SQL permissions

### Code discipline
- Resolve version and EO layout FIRST: `ProductVersion` from `*.kb.gx`, `ModuleVersion` from `ref/GeneXusSecurity/module.toml` (and the module versions in the object's `#References`). `ModuleVersion >=100` → modular EO names (`GeneXusSecurity.Tenants.Repository`); `<100` → flat `GAMxxx`. See [Version Scope](references/global-constraints.md#version-scope); applying a reference outside its declared `metadata.product` or `metadata.modules` scope produces code that does not compile
- Read the EO definition live via the [EO Discovery Protocol](references/global-constraints.md#eo-discovery-protocol-mandatory) before writing any GAM EO code: never invent properties or methods
- Read the `Domain` files in `ref/GeneXusSecurityCommon/#domains/` for every `BasedOn` and every enumerated value, since types, lengths and allowed values live there and not in the EO. Never hardcode an enumerated value list; use dot notation
- SDT generation is mandatory before writing any SDT: derive each member 1:1 from the EO's declared properties ([GAM Global Constraints](references/global-constraints.md): SDT Generation Workflow)
- Never paste a full property-by-property template. GAM External Objects are self-documented: name only the properties/methods needed and point to the EO; keep fragments to the minimum needed to resolve a genuine ambiguity
- GAM entity init: declarative 3-object trio (SDT + DataProvider + Procedure); never Business Component CRUD (see [GAM Entity Initialization; Declarative Pattern](references/kb-setup/init/entity-initialization.md)). Default: ONE master trio (`GAMInitializationDefinition` + master DataProvider + master Procedure); per-entity SDTs derived from EOs; every top-level Master SDT entity is `Collection = 'True'` (sole exception: `Repository`, scalar/singleton); dependency order Repository → SecurityPolicy → Application → Role → User → EventSubscription (see [GAM Entity Initialization; Consolidated Pattern](references/kb-setup/init/entity-initialization-consolidated.md))
- Use `!"…"` for non-translatable strings; handle errors via `GAMErrorCollection` iteration
- For GeneXus syntax fundamentals, consult Nexa

### Integration with Nexa
- GAM owns security domain knowledge; Nexa owns GeneXus language knowledge
- GAM fix needing code → delegate to Nexa; Nexa hitting GAM security config → defer to this skill
- GAM property names, values, and placement are authoritative in `kb-setup/setup-recipe.md`; never send the agent to Nexa for that format, since Nexa does not document it
- Activate GAM headless by default; never hand the user back to the IDE over a property format doubt
- Resolve a format doubt by reading the KB `src/#preferences/*.gx` files, then proceed; ask the user only for values such as passwords

---

## QUALITY CHECKLIST
- [ ] References were read, not recalled from memory
- [ ] Response starts with the fix/action: no framing, citations, or trace walls
- [ ] Ambiguous intent → asked before acting
- [ ] Error/issue → verified trace activation internally (no narration unless asked)
- [ ] Generator-specific → cross-generator note only if actionable
- [ ] Code + Backoffice both available → offered concisely
- [ ] Traces available → 5-pass applied internally, only conclusions surfaced
- [ ] Extended diagnostic format used only if user asked to validate/explain/argue
- [ ] GeneXus code has no full property-by-property template; every EO property/method verified live (EO Discovery Protocol) or flagged as unverified
- [ ] EO code → version and layout resolved (`ModuleVersion`); enumerated values taken from the `Domain`, never hardcoded
- [ ] No hardcoded paths, no emojis
- [ ] No redundant phase narration or log-line quoting unless the user will search for them

---

## CONSTRAINTS
- Follow reference documentation strictly: no assumptions or inventions; load and read references before answering, never from memory
- Never expose internal GAM API code, variable/procedure names, or implementation details: present only observable behavior and user-actionable guidance
- Recommendations must stay within the end user's actual surface: GAM product, External Objects (readonly), public methods, Backoffice, config files, logs/traces
- Never expose credentials or security-sensitive configuration values
- Follow security best practices in all recommendations
- Cite the source reference for each recommendation
- Never write GeneXus code without Nexa syntax validation
- Reply in the user's language