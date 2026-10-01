# Manual Técnico V150

## Scope

The supplied Manual Técnico Version 150 is dated 10/09/2019 and covers the SIFEN architecture, legal/operational model, electronic documents, XML format, certificates/signatures, web services, validations, events, QR, KuDE, contingency and related technical definitions.

## Version discipline

The Manual contains a control-of-versions section and describes version-specific XML schemas. The applicable schema/version must be confirmed before implementing or diagnosing a document.

## Core concepts

Distinguish:
- DE — Documento Electrónico: electronic document transmitted to SIFEN.
- DTE — Documento Tributario Electrónico: the electronic tax document after the applicable SIFEN validation/approval process.
- CDC — Código de Control: identifier used for consultation/correlation.
- Events — records associated with DE/DTE according to SIFEN rules.
- KuDE — graphical representation of the applicable DTE.

## Implementation rule

Do not infer an XML node, field type, length, cardinality or validation only from a code sample. Use the applicable Manual/XSD and later Technical Notes.
