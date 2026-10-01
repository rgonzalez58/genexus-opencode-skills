---
name: domain-cross-generator
description: Known differences between .NET Framework and Java generators in GAM
---

# GAM Cross-Generator Issues Encyclopedia
## Purpose
This document catalogs known behavioral differences between the GeneXus .NET Framework and Java generators that impact GAM flows. These differences are not bugs per se, but rather platform-specific behaviors that cause identical GAM logic to produce different results

---

## URL Encoding of `FromURL` in `APIStateClientPar`
Severity: Under Investigation (NOT CONFIRMED)
Affects: SLO with External IDP (Chained/IDP-of-IDP)
Generators: Java vs .NET

### Status
Initial hypothesis suggested Java stored `FromURL` URL-encoded. Testing with GAM 4.1.5 on Java/Tomcat 10.1 showed both generators store `FromURL` decoded:
```
GAMTrace-SLOProcess - LogoutType: 3   &APIStateClientPar.FromURL:http://localhost:8080/…/afterlogoutobject
```

The original customer's URL concatenation issue may stem from a different GAM version, configuration, or external IDP behavior

### How to Diagnose
Look for these traces in the SLO flow:

```
GAMTrace-SLOProcess - LogoutType: 3   &APIStateClientPar.FromURL:<URL>
GAMTrace-GAMExternalAuthenticationInputValidParam-&APIStateClientPar: {"FromURL":"<URL>"}
```

If `FromURL` contains `%3A`, `%2F`, etc. → encoding problem
If `FromURL` starts with `http://` or `https://` → correct

---

## `redirect_uri` Handling in Signout QueryString
Severity: Medium
Affects: SLO with GAMRemote
Generators: Both (logging artifact)

### Observation
In the `AbsoluteUri dynamicport` log line, the `redirect_uri` parameter may appear with `%` signs stripped in the logged URL (e.g., `http3A2F2F` instead of `http%3A%2F%2F`). This is a logging artifact -- the actual HTTP processing handles the encoding correctly. Do not diagnose based on the `AbsoluteUri dynamicport` log line; use the `GAMTrace-GAMExternalAuthenticationInputLoadParam: HTTPRequest QueryString:` line instead

---

## Checklist: Analyzing a GAM Issue Across Generators
When a GAM flow works in one generator but fails in another:

- Compare the `APIStateClientPar` JSON stored in DB -- look for encoding differences in URL fields
- Compare the `LoadParam` traces -- check if the parsed `&aParameters` array produces the same values
- Compare the `RedirToURL OK:` traces -- the final redirect URL should be identical
- Check `Link()` vs `response.sendRedirect()` behavior -- .NET and Java handle relative vs absolute URLs differently
- Verify virtual directory URL adjustment -- GAM adjusts URLs based on the current environment's virtual directory; this behavior may differ across generators

---

## `IDP-` State Prefix Regression (u13HF/u14)
NOTE: This is a version regression (affects .NET and Java equally), not a cross-generator difference. Canonical reference: [GAM Session and Token Reference](../authentication/external-providers/common/domain-session-token.md) — section "CRITICAL: IDP- Prefix Mechanism"

Summary: When IDP and Client share the same GAM database, GAM u13HF and u14 fail with PRIMARY KEY violation on the shared state table. Fixed in u14 HotFix. Workaround: separate GAM databases per application
