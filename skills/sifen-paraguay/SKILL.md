---
name: sifen-paraguay
description: Expert skill for Paraguay SIFEN/e-Kuatia electronic invoicing, Manual Técnico V150, XML/XSD, digital signatures, web services, asynchronous lots, validation, CDC, events, QR, KuDE, contingency, testing, and production diagnostics. Use when analyzing, designing, implementing, reviewing, or troubleshooting SIFEN integrations, especially Node.js/GeneXus projects.
metadata:
  version: 1.0.0
  author: Custom project skill
  language: es
---

# SIFEN Paraguay — Expert Skill

## Purpose

Provide implementation and diagnostic guidance for Paraguay's SIFEN/e-Kuatia electronic invoicing system, prioritizing the official DNIT technical documentation supplied with this skill.

This skill is designed to work together with:
- `nodejs` for Node.js architecture, async processing, HTTP, workers, logging, PDF and email.
- `sqlserver` for SQL Server transactions, blocking, batches, retries and diagnostics.
- `nexa` for GeneXus objects, syntax, KB operations and GeneXus-specific implementation.
- `gam` only when the problem is specifically GAM/security-manager related.

## Source hierarchy

Always distinguish these layers:

1. **OFICIAL DNIT/SIFEN** — Manual Técnico V150, Technical Notes, XSD/XML structures, test guide and DNIT recommendations.
2. **PATRÓN GNB** — the supplied GNB project example. It is an implementation reference, not an official SIFEN rule.
3. **PROYECTO ACTUAL** — facturapy/iCopyFE or another project supplied by the user.
4. **INFERENCIA/BUENA PRÁCTICA** — engineering recommendations that are not explicitly required by DNIT.

Never convert a GNB implementation detail into an official SIFEN requirement.

When official documentation and project code disagree, identify the difference explicitly and prioritize the applicable official DNIT rule.

## Core rules

- Identify the SIFEN version before changing XML, XSD, service URLs, validations or business rules.
- Treat Technical Notes as amendments/corrections to the Manual Técnico and check them before relying on an older rule.
- Use the XSD as a structural validation source; do not invent XML nodes or attributes.
- Preserve exact XML element names and case.
- Do not silently omit mandatory fields.
- Do not add formatting whitespace/comments/prefixes or empty elements when the applicable SIFEN documentation prohibits them.
- Never assume that a successful HTTP/SOAP response means that a DE was approved.
- Separate:
  - transport/connection result,
  - reception result,
  - asynchronous lot processing result,
  - DE validation result,
  - DTE approval,
  - event result.
- For asynchronous lots, persist enough state to correlate the submitted lot with its returned lot number/CDC and later consultation.
- Design retries to avoid duplicate submissions. A timeout or ECONNRESET does not prove that SIFEN did not receive the request.
- For uncertain reception, use the applicable SIFEN consultation mechanism before blindly resending.
- Keep logs and audit/control records intact when troubleshooting.
- For batch processing, prefer bounded concurrency and explicit state transitions.
- Never recommend `NOLOCK`, disabling integrity, deleting audit logs, or blindly resending documents as a generic SIFEN fix.
- Never expose private keys, passwords, P12/PFX contents, CSC secrets or certificate credentials in logs.

## SIFEN workflow

Typical flow:

1. Build DE XML according to the applicable version and XSD.
2. Validate structure and business prerequisites.
3. Sign the DE using the required digital-signature mechanism.
4. Generate/prepare QR data when applicable.
5. Send DE individually or as an asynchronous lot according to the required service.
6. Record the immediate SIFEN response.
7. If the service is asynchronous, query the lot result separately.
8. Process each DE result.
9. Persist CDC/status/error information.
10. For approved DTEs, generate/deliver KuDE as required.
11. Register and process applicable events.
12. Support later CDC/DTE/event consultation and audit.

## Asynchronous lots

The supplied DNIT best-practices document states that lots can contain up to 50 DE and are processed asynchronously. The result must be consulted separately from submission.

Important documented reception codes:
- `0300`: lot received successfully and queued for processing; consult the returned lot identifier.
- `0301`: lot was not queued; investigate the documented reason before retrying.

The best-practices guide recommends sending the maximum possible number of documents per lot, up to 50, and avoiding premature repeated queries. It also describes temporary blocking conditions and recommends appropriate intervals between consultations.

Do not turn these values into hard-coded application rules without checking the version of the official documentation currently applicable to the project.

## Diagnostics

When the user reports a failure, classify it first:

### A. XML/XSD
Symptoms:
- schema validation error
- missing/extra node
- wrong type/length
- namespace/version mismatch

Action:
- identify the exact node and applicable XSD/manual section;
- compare generated XML against the official structure;
- check Technical Notes.

### B. Signature/certificate/TLS
Symptoms:
- certificate error
- mutual TLS failure
- signature invalid
- certificate revoked/expired
- TLS handshake failure

Action:
- separate transport authentication from XML digital signature;
- verify certificate chain, validity, private-key access and environment;
- never log secrets.

### C. HTTP/SOAP/service
Symptoms:
- timeout
- ECONNRESET
- connection refused
- HTTP 4xx/5xx
- SOAP fault

Action:
- capture endpoint, environment, timestamp, timeout, HTTP/SOAP response and correlation data;
- determine whether SIFEN may have received the request before retrying;
- do not equate transport failure with business rejection.

### D. Asynchronous lot
Symptoms:
- lot accepted but result pending
- lot not queued
- individual DE rejected after lot reception
- repeated consultation

Action:
- persist lot identifier;
- track state transitions;
- query according to documented timing;
- separate lot state from individual DE state.

### E. Business validation
Symptoms:
- SIFEN returns validation/rejection codes.

Action:
- preserve code/message;
- map it to the applicable manual/Technical Note;
- identify the XML field/group involved;
- fix generation logic, not only the response handling.

### F. QR/KuDE
Treat QR validation and KuDE rendering as downstream concerns. A visually correct PDF does not prove that the DTE is approved, and a generated QR does not replace SIFEN validation.

## Testing

Use the official test guide as the baseline for test coverage. It describes testing of:
- mutual authentication/communication;
- DE transmission and validation/rejection;
- events;
- DTE and event consultation;
- KuDE generation/transmission;
- error scenarios;
- QR validation.

Do not declare a SIFEN implementation complete based only on one successful invoice.

## GNB implementation reference

The supplied GNB example contains:
- controllers for XML generation, lot sending and consultation;
- services for lot processing, consultation workers, state updates, documents and email;
- iSeries/database access;
- worker-pool utilities;
- daemon executables;
- workaround scripts for regeneration and mass consultation.

Use this architecture only as an implementation pattern. In particular, GNB demonstrates separation of generation, sending, asynchronous consultation, state update and delivery concerns.

## Node.js implementation

For Node.js:
- delegate generic Node architecture to `nodejs`;
- keep SIFEN-specific rules here;
- use bounded workers for lots/consultations;
- use explicit idempotency and state transitions;
- keep request timeout/retry policy separate from SIFEN business status;
- persist raw/diagnostic responses where appropriate, without secrets;
- use PM2 or the project's established process manager when appropriate;
- do not introduce Redis/Bull merely because a queue exists conceptually.

## GeneXus implementation

For GeneXus:
- delegate GeneXus syntax/object work to `nexa`;
- this skill owns SIFEN rules, XML semantics, service behavior and integration requirements;
- if embedded Java is required, preserve the project's GeneXus Java syntax conventions and let `nexa` validate the actual GeneXus syntax.

## Security

Never print:
- private keys;
- P12/PFX passwords;
- CSC secrets;
- access tokens;
- full certificate/private-key material.

When showing XML or SOAP examples, redact credentials and sensitive identifiers.

## Output style

When solving a SIFEN problem:
1. State the likely category of the problem.
2. State the documented rule/evidence.
3. Explain the impact on the current implementation.
4. Give the smallest safe change.
5. Provide validation steps.
6. Clearly label whether a statement is `OFICIAL DNIT`, `PATRÓN GNB`, `PROYECTO ACTUAL`, or `RECOMENDACIÓN`.

Do not fabricate a SIFEN code, field, endpoint, validation or rule.

## References

Load only the references relevant to the question. Prefer focused references over reproducing the full Manual Técnico.

- `references/official/source-map.md`
- `references/official/manual-v150.md`
- `references/official/best-practices.md`
- `references/official/test-guide.md`
- `references/official/technical-note-026.md`
- `references/official/technical-note-027.md`
- `references/services/asynchronous-lots.md`
- `references/services/diagnostics.md`
- `references/xml/xml-generation.md`
- `references/security/signature-tls.md`
- `references/testing/test-matrix.md`
- `references/implementation/gnb-pattern.md`
- `references/implementation/nodejs-pattern.md`
- `references/implementation/project-boundaries.md`
