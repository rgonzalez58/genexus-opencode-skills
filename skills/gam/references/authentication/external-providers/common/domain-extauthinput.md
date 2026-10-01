---
name: domain-extauthinput
description: GAM External Authentication entry point — routing behavior, OriginType meanings, trace signatures
---

# GAM External Authentication Entry Point
## DEFINITION
This is the single entry point where GAM processes all external authentication callbacks, cross-repository communications, and SLO (Single Log-Out) requests. Understanding how this routing works is essential for diagnosing why a request enters the wrong flow or fails silently

---

## Processing Pipeline
Every request arriving at this entry point passes through five stages:

- Trace Initialization -- If tracing is enabled, all subsequent log entries appear prefixed with `GAMTrace-GAMExternalAuthenticationInput`
- Parameter Loading -- The incoming QueryString/Body is parsed into parameters. Two key values are extracted: `OriginType` and `OriginTypeValue`
- Parameter Validation -- The request is validated and the authentication type is determined. GAM also decides whether the current node is acting as an Identity Provider (IDP) or as a Client
- Execution Routing -- Based on the IDP/Client determination, GAM routes to the appropriate behavior (see "Routing When Acting as an IDP" and "Routing When Acting as a Client" below)
- Response Building -- GAM constructs the final response: either an HTTP 302 redirect or a JSON status code response

---

## Routing When Acting as an IDP
When the current node is the Identity Provider, GAM routes based on `OriginTypeValue`:

-	OriginTypeValue `auth` -- A client application is requesting authentication
	*	If the IDP is an "IDP-of-IDP" (federated), GAM saves the client request state and redirects to the upstream IDP, calculating valid scopes for the application
	*	If it is a standard IDP, GAM presents the sign-in step
	*	Authentication is then delegated to the appropriate OAuth 2.0 or GAM Remote handler
-	OriginTypeValue `signout` -- A client is requesting Single Log-Out
	*	GAM initiates the SLO flow by delegating to the GAM Remote sign-out handler
-	OriginTypeValue `expire` -- A token expiration notification
	*	GAM kills the local sessions associated with the specified token

---

## Routing When Acting as a Client
When the current node is NOT the IDP (it is receiving a callback or acting as a service consumer), GAM routes based on both `OriginType` and `OriginTypeValue`:

-	OriginType `miniapp` -- A mini-app authentication callback
	*	GAM processes the mini-app authentication. If successful, it creates a local session
-	OriginType `cache` / OriginTypeValue `all` -- Cache clear request (full)
	*	GAM clears the entire node cache. Returns HTTP 200
-	OriginType `cache` / OriginTypeValue `rep` -- Cache clear request (repository)
	*	GAM clears the cache for the specific repository. Returns HTTP 200
-	OriginType `signout` (from external OAuth IDP) -- SLO callback from an external OAuth 2.0 provider
	*	GAM expires the local token and session
-	OriginType `signout` (from GAM Remote) -- SLO callback from a GAM Remote IDP
	*	GAM delegates to the GAM Remote sign-out handler
-	Default (return from IDP login) -- User returning after successful authentication at an external IDP
	*	GAM enters the "Login from External IDPs" flow (see below)

### Login from External IDPs
When a user successfully authenticates at an external IDP (Google, Facebook, Apple, GAM Remote, SAML 2.0, OAuth 2.0) and is redirected back to GAM, the following behavioral sequence occurs:

- Provider Identification -- GAM identifies which external provider the user authenticated with and invokes the corresponding provider-specific handler
- User Provisioning -- GAM checks whether the user already exists locally. If auto-registration is enabled and the user does not exist, GAM creates them. If the user exists, GAM updates their data from the IDP response
- Local Session Generation -- GAM creates a local session. The session type (OAuth or Standard) determines the exact session creation path used
- Event Execution: GAM fires the `ExternalAuthentication_Response` event subscription, passing the raw IDP JSON response so that custom GeneXus procedures can read the IDP claims. The handler deserializes it into `ExternalAuthenticationResponseProperties`, see [GAM Events Subscription: Handler Contract](../../../events/domain-event-subscription.md)

---

## Response Generation
The final stage determines how control is returned to the caller:

-	Cache, MiniApp, or Expire requests
	*	HTTP status code (200 OK, 417 Error, or 401 Unauthorized) with a JSON payload
-	OAuth sessions returning to a frontend client
	*	HTTP 302 redirect with `state`, `access_token`, and `expires_in` as URL fragments to the `redirect_clientloginurl`
-	Standard browser sessions
	*	HTTP 302 redirect to the configured redirect URL (or fallback URL)
-	No redirect URL available (browser flow)
	*	HTTP 417 with a JSON error payload

---

## Trace Signatures
When debugging logs containing `GAMExternalAuthenticationInput`, look for:

-	`GAMTrace-GAMExternalAuthenticationInput Parm:`
	*	The raw URL-encoded parameters received from the caller
-	`GAMTrace-GAMExternalAuthenticationInput ==ValidWhenIsIDP-True ++++++`
	*	The node is acting as an IDP in this request
-	`GAMTrace-GAMExternalAuthenticationInput ==ValidWhenIsIDP-False ++++++`
	*	The node is acting as a Client in this request
-	`GAMTrace-GAMExternalAuthenticationInput ==LoginFromExternalIDPs ++++++`
	*	The user is returning from an external IDP (Apple/Google/SAML/OAuth)
-	`GAMTrace-GAMExternalAuthenticationInput &RedirToURL OK:`
	*	The exact final URL GAM is about to redirect the user to. Very useful for catching URL encoding or concatenation bugs
