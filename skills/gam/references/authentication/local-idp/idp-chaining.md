---
name: idp-chaining
description: GAM IdP as a transparent login proxy — auth: Local Login URL auto-redirects a client to an OAuth 2.0/GAMRemote auth type, chains IdP to IdP, and returns to the originating client
---

# IDP→IDP Login Chaining — `auth:` Local Login URL Proxy
How a GAM Identity Provider redirects an incoming client straight to another authentication type (an external OAuth 2.0 IdP or another GAM IdP via GAMRemote) without showing its own login page, how those redirects chain across IdP nodes, and how control returns to the client that started the login. Also called IdP-of-IdP or chained login

This is the LOGIN direction. For the logout direction of the same chain, see [SLO External IDP Analysis Playbook](../../logout/domain-slo-external-idp.md)

Related files:
- [GAM as SP — GAMRemote (Web Flow) Initialization](../local-sp/domain-gamremote.md) — the GAMRemote auth type used as a chain hop (its `RemoteServerURL` is the next IdP)
- [GAM as Web IDP Server — Authorization Code Flow](authorization-code-flow.md) — non-GAM/custom client (React, mobile) against a GAM Web IDP: signin/callback, `state`, PKCE, custom callback URL
- [GAM Authentication Configuration Reference](../external-providers/common/domain-auth-config.md) — external OAuth 2.0/OIDC auth-type configuration (terminal hop)
- [GAM Backoffice — Applications (GAM_Applications)](../../backoffice/applications.md) — `Local Login URL` field mapping (`ClientLocalLoginURL`)
- [GAM Backoffice — Authentication Types (GAM_Authentication_Types)](../../backoffice/auth-types.md) — `Redirect to Authenticate` flag on the target auth type
- [GAM-as-SP — OAuth 2.0 Common Flow (observable in trace)](../external-providers/oauth20/common/oauth20-common-flow.md) — `state` lifecycle and callback machinery

Official documentation: [Identity Provider Configuration for GAM Remote Authentication](https://docs.genexus.com/en/wiki?37038,Identity+Provider+Configuration+for+GAM+Remote+Authentication)

---

## Concept
A client application registered at a GAM IdP has a `Local Login URL` property (`GAMApplication.ClientLocalLoginURL`) — a mandatory field whose value is normally a login object URL, `GAMExampleIDPLogin` for a GAM acting as IDP (see [Local Login URL — `GAMExampleIDPLogin`](authorization-code-flow.md#local-login-url--gamexampleidplogin)). Instead, it can be an `auth:` directive that tells the IdP to skip that login page and redirect the browser straight to a configured authentication type:

```
auth:<authtype>[;<authtype>…]
```

- The IdP auto-redirects to the named authentication type — it acts as a transparent login proxy; the originating client never sees the intermediary IdP's login page
- Supported target types: OAuth 2.0 and GAMRemote
- Each target must exist in the IdP repository and have `Redirect to Authenticate = True` — see [GAM Backoffice — Authentication Types (GAM_Authentication_Types)](../../backoffice/auth-types.md)
- `;` lists ALTERNATIVE auth types, not a sequence. GAM picks one: the type named by the request `authentication_type_name`, or the first when only one is listed

Official example (`GAMApplication` EO):

```genexus
&GAMApplication.ClientLocalLoginURL = !"auth:oauth20-google;gamremote"
```

## Topology
```
Client A (GAM)  ──┐
Client B          │ login   ┌─────────────┐  auth:gamremote   ┌─────────────┐  auth:oauth20-google  ┌────────────┐
(non-GAM, uses    ├────────▶│ IdP 1 (GAM) │──────────────────▶│ IdP 2 (GAM) │──────────────────────▶│ Google ext │
GAMRemote svcs) ──┘         │  auth:…     │◀──── return ───────│  auth:…     │◀───── return ─────────│ OAuth 2.0  │
                            └─────────────┘                    └─────────────┘                       └────────────┘
                                   ▲                                                                        │
                                   └──────────────── unwind: return to Client A home object ───────────────┘
```

- Each GAM node owns its own `Local Login URL` (`auth:…`) and its own target auth types
- A `gamremote` token resolves to a GAMRemote auth type whose `RemoteServerURL` is the NEXT IdP
- The chain length is bounded only by configuration — every hop repeats the same pattern
- Login starts and ends at the SAME client: the final IdP authenticates, then control unwinds hop by hop back to the originating client's home object

## Client types — who starts the login
The `auth:` proxy is transparent to the client: the client only talks to the FIRST IdP and is unaware of the hops beyond it. Both client kinds start a chain the same way:

- GeneXus app WITH GAM — authenticates via a GAMRemote auth type pointing at the first IdP. GAM drives the redirect and callback automatically. Configure per [GAM as SP — GAMRemote (Web Flow) Initialization](../local-sp/domain-gamremote.md)
- Non-GAM / custom client (React SPA, mobile, any backend) — no GAM module. It calls the first IdP's Web IDP endpoints itself (`/oauth/gam/signin`, callback, `access_token`, `userinfo`) and validates `state` on its own — see [GAM as Web IDP Server — Authorization Code Flow](authorization-code-flow.md). It MUST be registered at the first IdP as an Application with `Custom callback URL? = True` and a matching `Callback URL`
- For public clients (React SPA, mobile) enable PKCE on the registration — see [PKCE (Proof Key for Code Exchange)](authorization-code-flow.md#pkce-proof-key-for-code-exchange)

## Configure a chain link
Every IdP node is configured the same way for the links it owns: its incoming-client Applications (4.1) and its outbound auth types (4.2 / 4.3). Each is settable by Backoffice or by code

### Incoming-client Application (the `auth:` proxy)
This is what makes a node redirect onward instead of showing its own login

- Backoffice: `Applications → Edit <client-app> → OAuth Authentication tab` → set `Local Login URL = auth:oauth20-google;gamremote`; fill `Callback URLs`; check `Custom callback URL?` for non-GAM/custom clients — see [GAM Backoffice — Applications (GAM_Applications)](../../backoffice/applications.md)
- By code: `&GAMApplication.ClientLocalLoginURL = !"auth:oauth20-google;gamremote"` — full init pattern in [GAM Entity Initialization — Declarative Pattern](../../kb-setup/init/entity-initialization.md); consult the Nexa skill for GeneXus syntax
- Each `auth:` token must name an OAuth 2.0/GAMRemote auth type that exists in this repository with `Redirect to Authenticate = True` — see [GAM Backoffice — Authentication Types (GAM_Authentication_Types)](../../backoffice/auth-types.md)

### Outbound hop to the next GAM IdP (GAMRemote)
- A GAMRemote auth type whose `RemoteServerURL` is the next IdP — configure by Backoffice or by code per [GAM as SP — GAMRemote (Web Flow) Initialization](../local-sp/domain-gamremote.md)
- The next IdP repeats 4.1 (its own incoming-client Application with `auth:…`) to continue the chain

### Outbound hop to an external IdP (OAuth 2.0 / OIDC) — terminal
- An OAuth 2.0 auth type for Google/Microsoft/Okta/… with `Redirect to Authenticate = True` — configure by Backoffice or by code per [GAM Authentication Configuration Reference](../external-providers/common/domain-auth-config.md) and [OAuth 2.0 Auth Type — Programmatic Initialization](../external-providers/oauth20/provider-generic-oauth20.md)
- This is the last hop; after it authenticates, the chain unwinds back toward the originating client

## Return path (unwind to the originating client)
Each hop carries a `state` value and returns to the IdP's own `/oauth/gam/callback`. The IdP restores the per-`state` context, then redirects to the stored return URL (`FromURL`) — the previous hop. This repeats backward until the originating client receives the final redirect and lands on its home object

- The client that STARTS the login is the client that ENDS it — the chain is symmetric
- `state` must survive every hop; a hop that drops or rewrites `state` breaks the return
- For the `state` lifecycle and callback machinery see [Step 1 — Signin (Redirect to IDP)](authorization-code-flow.md#step-1--signin-redirect-to-idp) and [Custom Callback URL](authorization-code-flow.md#custom-callback-url) in `authorization-code-flow.md`, and [GAM-as-SP — OAuth 2.0 Common Flow (observable in trace)](../external-providers/oauth20/common/oauth20-common-flow.md)

## Relationship to SLO
Login chaining and logout chaining traverse the same node graph and both return to the originating client, but they use DIFFERENT mechanisms:

- LOGIN — driven by the `auth:` Local Login URL on each client application (this file)
- LOGOUT (SLO) — driven by the session hierarchy (parent/daughter sessions) and `GAMRemoteLogoutBehavior`, NOT by `auth:`

Do not expect an `auth:`-style logout directive. To propagate logout across the whole chain, set `GAMRemoteLogoutBehavior = All` at the repository and configure SLO per [SLO External IDP Analysis Playbook](../../logout/domain-slo-external-idp.md)

## Diagnostic intake
Before diagnosing a broken login chain, gather:

- Where does login START — which client application, and is it GAM or non-GAM/custom (React, mobile)?
- WHO renders the error — the originating client, an intermediary IdP, or the final IdP?
- At WHICH hop does the flow stop (browser URL at the moment of failure)?
- Are the signin parameters correct — `client_id`, `redirect_uri`, `state`, `scope`, and `authentication_type_name` when several alternatives are listed?
- For custom/non-GAM clients: is the application registered with `Custom callback URL? = True` and a `Callback URL` that matches the redirect?
- Whose traces do we have — each GAM node logs only its own hop; non-GAM clients produce no GAM traces

## Login-direction failure catalog
- Condition: `auth:` token does not match any auth type name
	* Trace signature: signin reaches the IdP but no redirect to the target is issued; auth-type lookup yields nothing
	* Symptom: IdP shows its own login page (proxy did not engage) or returns an error to the client
	* Root cause: typo or case mismatch between the token and the auth type `Name`, or the type is not OAuth 2.0/GAMRemote
	* Fix: make each `auth:` token match an existing OAuth 2.0/GAMRemote auth type name exactly

- Condition: target auth type not set to `Redirect to Authenticate`
	* Trace signature: login starts but no outbound redirect to the IdP/provider is emitted — see [No authorize redirect — login page stays](../../debugging/common-login-issues.md#no-authorize-redirect--login-page-stays)
	* Symptom: login page stays; no auto-redirect
	* Root cause: `RedirectToAuthenticate = False` on the target type
	* Fix: enable `Redirect to Authenticate` on the target auth type (Backoffice → Auth Types)

- Condition: custom/non-GAM client without `Custom callback URL?`
	* Trace signature: callback rejected, or the redirect is built against the auto `/oauth/gam/callback` the client does not serve
	* Symptom: error after the target IdP authenticates; client never receives the callback
	* Root cause: GAM auto-builds `<BaseURL>/oauth/gam/callback`; a custom client serves a different endpoint
	* Fix: check `Custom callback URL?` and register the client's real `Callback URL`

- Condition: `redirect_uri` not in the application's `Callback URLs`
	* Trace signature: signin refused with an invalid-redirect error before any provider redirect
	* Symptom: error answered by the IdP that owns that client registration
	* Root cause: the requested `redirect_uri` is not whitelisted
	* Fix: add the exact `redirect_uri` to `Callback URLs`

- Condition: `authentication_type_name` missing with multiple alternatives
	* Trace signature: IdP redirects to the FIRST listed type, not the intended one
	* Symptom: user lands at the wrong provider
	* Root cause: `;` lists alternatives; absent `authentication_type_name`, GAM picks the first
	* Fix: pass `authentication_type_name` on signin to select the target

- Condition: intermediary IdP unreachable (GAMRemote hop)
	* Trace signature: the hop's `RemoteServerURL` request times out or returns a connection error
	* Symptom: chain stops at the node before the unreachable IdP
	* Root cause: wrong `RemoteServerURL`, network/DNS, or the next IdP is down
	* Fix: verify `RemoteServerURL` and the network path between the two GAM servers

- Condition: `state` lost on a return hop
	* Trace signature: a callback arrives without `state`; the node cannot restore context
	* Symptom: user redirected to login instead of continuing the unwind
	* Root cause: a hop dropped or rewrote `state`, or the context TTL expired mid-flight
	* Fix: confirm every hop preserves `state`; ensure the round trip completes within the context TTL
