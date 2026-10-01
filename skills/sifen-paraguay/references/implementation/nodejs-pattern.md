# Node.js SIFEN implementation pattern

Delegate generic Node.js decisions to the `nodejs` skill. This file only defines SIFEN-specific boundaries.

Recommended components:
- SIFEN client for SOAP/HTTP;
- XML generator/validator;
- signature service;
- lot builder;
- lot submission worker;
- lot consultation worker;
- result parser;
- state persistence;
- QR/KuDE pipeline;
- audit logger.

## Worker principle

Keep submission and consultation independently controllable. Consultation should not block invoice generation.

## State principle

Persist explicit states rather than deriving everything from log text.

## Retry principle

Retry transport failures with bounded attempts, but first consider duplicate submission risk and SIFEN consultation.

## Integration

For SQL Server, delegate database locking/transaction analysis to `sqlserver`.
For GeneXus, delegate object/code syntax to `nexa`.
