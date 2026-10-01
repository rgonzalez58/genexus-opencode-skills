# Asynchronous services and lot lifecycle

## Concept

A lot submission and a lot-result consultation are separate operations.

Conceptual state flow:

SUBMITTED
→ RECEIVED/QUEUED
→ PROCESSING
→ COMPLETED
→ per-DE APPROVED / REJECTED / OTHER DOCUMENTED RESULT

The exact SIFEN response/state values must come from the applicable Manual/XSD and current technical documentation.

## Minimum persisted correlation

For each lot, retain:
- internal batch identifier;
- SIFEN lot identifier when returned;
- submission timestamp;
- environment;
- sender RUC;
- number of DE;
- request/response diagnostic data without secrets;
- current lot state;
- last consultation timestamp;
- consultation attempt count;
- individual DE result;
- CDC when available;
- error/validation codes.

## Duplicate prevention

A timeout does not prove non-reception. Before retrying a submission, determine whether the original request may have been accepted and whether consultation can resolve its status.

## Concurrency

Use bounded workers. Do not issue unbounded Promise.all over thousands of lots.
