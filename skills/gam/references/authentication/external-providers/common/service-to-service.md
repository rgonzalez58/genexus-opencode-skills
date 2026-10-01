---
name: service-to-service
description: Service-to-Service authentication with GAMAgentServiceHeader for propagating the security context between backend services
---

# Service-to-Service Authentication (`GAMAgentServiceHeader`)
Related files:
- [GAM Authentication](domain-auth.md) — Main authentication flows
- [GAM as Web IDP Server — Authorization Code Flow](../../local-idp/authorization-code-flow.md) — Web IDP flow when the receiver is GAM-protected
- [GAM as REST IDP Server — Password Grant Flow](../../local-idp/password-grant-flow.md) — REST IDP flow for direct service-to-service calls
- [GAM Backoffice — Applications (GAM_Applications)](../../../backoffice/applications.md) — Application trust settings

When a backend service needs to call another backend service in a secured GAM environment, it must propagate the security context. GAM provides the `GAMAgentServiceHeader` mechanism for this

## Behavior
- The calling service invokes `GAM.GetAgentServiceHeader()` to obtain a header name-value pair
- The header value is resolved from the current authenticated security context
- The calling service injects this header into the outbound HTTP request
- The receiving service validates the header through its own GAM middleware

## Usage Pattern
```genexus
// Get the security header for service-to-service calls
&AgentServiceHeader = GAM.GetAgentServiceHeader()

// Verify the header was resolved
If &AgentServiceHeader.Name.IsEmpty() or &AgentServiceHeader.Value.IsEmpty()
	// No security context available — caller may not be authenticated
	Msg(!"Agent header unavailable")
	Return
EndIf

// Inject the header into the outbound call
&HttpClient.AddHeader(&AgentServiceHeader.Name, &AgentServiceHeader.Value)
&HttpClient.Execute(!"GET", &TargetServiceUrl)
```

## `GAMAgentServiceHeader` Properties
- `Name`: header name to send in outbound calls
- `Value`: header value resolved for the current security context

## Constraints
- Header value depends on the current authenticated security context — if no session is active, the header is empty
- Use only for trusted service-to-service calls (backend to backend)
- Both services must share the same GAM Repository or be configured for cross-repository trust
- The receiving service must have GAM security middleware enabled to validate the header
