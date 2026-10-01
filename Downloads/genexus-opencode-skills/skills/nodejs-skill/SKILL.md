---
name: nodejs
description: Node.js expert skill for backend services, APIs, asynchronous processing, database integration, batch workloads, workers, PM2, logging, PDF generation, email delivery, XML/QR processing, error handling, performance, and production diagnostics. Use for Node.js development and troubleshooting, especially Node.js applications integrated with SQL Server, DB2, Oracle, GeneXus, and external services.
metadata:
  version: 1.0.0
  author: Custom
  domain: nodejs
---

# Node.js Expert

## Purpose

Provide practical, production-oriented guidance for Node.js applications, with emphasis on reliability, asynchronous execution, database access, background jobs, batch processing, external services, and Windows deployments.

This skill is complementary to other specialized skills:
- `sqlserver` owns SQL Server diagnosis and optimization.
- `nexa` owns GeneXus language, objects, and KB workflows.
- `gam` owns GeneXus Access Manager and security diagnostics.

## Activation

Use this skill for:
- Node.js application architecture
- CommonJS/ESM compatibility
- async/await, Promises, concurrency and error handling
- Express/API services
- SQL Server, DB2 and Oracle connectivity
- connection pools
- batch processing and workers
- PM2 deployment and process management
- Windows production services
- Winston or structured logging
- PDF generation with Puppeteer
- Nodemailer and email jobs
- XML generation/signing/parsing
- QR generation
- SIFEN/electronic invoicing integrations
- Axios/HTTP integrations
- retries, timeouts, and external-service failures
- memory, CPU, event-loop and process diagnostics
- graceful shutdown
- environment configuration and secrets
- filesystem handling and invoice storage

Do not use this skill as the primary skill for:
- SQL Server query/index/locking diagnosis: delegate to `sqlserver`
- GeneXus object modeling or syntax: delegate to `nexa`
- GAM authentication/authorization: delegate to `gam`
- generic frontend-only JavaScript

## Core principles

1. Diagnose before changing production code.
2. Prefer simple, observable, maintainable asynchronous flows.
3. Do not block the event loop with avoidable CPU-heavy synchronous work.
4. Bound concurrency. Do not replace a serial bottleneck with an uncontrolled Promise storm.
5. Reuse database connection pools instead of opening a connection per record.
6. Define timeouts for external HTTP calls.
7. Retries must be bounded and safe to repeat.
8. Treat duplicate processing and partial failure as normal possibilities.
9. Use idempotent processing where possible.
10. Keep credentials and secrets out of source code.
11. Log enough to diagnose failures without exposing passwords, tokens, or private keys.
12. Separate business state from technical processing state.
13. Implement graceful shutdown for API and workers.
14. Do not claim performance improvements without measurement.
15. Never invent package APIs, version compatibility, or configuration behavior.

## Production architecture

Typical backend:

```text
HTTP/API
  |
  +-- validation
  +-- business service
  +-- database/repository
  +-- external service
  |
  +-- background worker
       +-- batch
       +-- PDF
       +-- email
       +-- XML/QR
```

Keep long-running work out of synchronous HTTP requests when business requirements allow asynchronous processing.

For Windows deployments using PM2, treat each worker/API as independently observable.

## Async and concurrency

Prefer `async/await` with explicit error handling.

Avoid uncontrolled:

```js
await Promise.all(milesDeRegistros.map(procesar));
```

For large workloads use bounded concurrency or explicit batches.

Choose concurrency based on CPU, memory, database pool size, external service limits, file I/O, and business rate limits.

## Database integration

Use connection pools and release resources correctly.

A Node.js problem can be caused by:
- exhausted pool
- leaked connection
- long transaction
- slow SQL
- excessive concurrent queries
- retry storm
- network timeout

When the root cause is SQL Server locking, waits, indexes, execution plans, or transaction design, use the `sqlserver` skill.

Do not solve database problems by blindly increasing pool size.

## Batch processing

A robust batch process defines:
- batch size
- concurrency
- transaction boundary
- progress state
- retry policy
- failure state
- idempotency
- logging
- recovery behavior

For large invoice workloads, process bounded lots and persist progress so a restart does not reprocess everything.

## Background jobs

A worker should:
1. discover pending work
2. claim work safely
3. process it
4. persist success/failure
5. release resources
6. log the outcome

Database flags/status fields are valid when already part of the business design.

Do not introduce Redis/Bull solely because they are common. First evaluate whether PM2 plus database-backed state satisfies the workload.

## PM2

PM2 is appropriate for API processes, independent workers, restart management, logging, and startup management.

Avoid running the same exclusive job in multiple instances unless it has a safe claim/locking mechanism.

## Error handling

Do not silently swallow errors.

Distinguish:
- timeout
- connection reset
- DNS failure
- HTTP 4xx
- HTTP 5xx
- malformed response
- application/business rejection

Each can require different handling.

## External HTTP services

Consider:
- connection timeout
- response timeout
- retry policy
- retryable status codes
- idempotency
- exponential backoff
- correlation ID
- response validation

For SIFEN and similar systems, distinguish transport failures from business acceptance/rejection.

## Files and PDFs

PDF generation can be CPU and memory intensive. Asynchronous syntax does not make CPU-heavy rendering non-blocking.

For large HTML/PDF jobs:
- avoid unnecessary copies of large strings
- use explicit rendering timeouts
- clean temporary files
- validate directories and paths
- monitor CPU and memory
- separate database persistence from physical-file generation when business rules allow it

## Email

For background email jobs:
- use a reusable transporter where appropriate
- process bounded batches
- record success/failure
- avoid duplicate sends
- verify attachments exist
- distinguish transient SMTP/network failures from permanent failures

Keep email delivery outside a critical database transaction whenever possible.

## XML, signing, QR and SIFEN

Typical flow:

```text
business data
   -> XML
   -> validation
   -> signing
   -> QR
   -> submission
   -> response/query
   -> PDF/KuDE
   -> email
```

Persist enough state to resume after failure.

Do not mark a document successfully submitted before the external operation justifies it.

Keep certificates and private keys outside source code.

## Logging

Prefer structured logs.

Recommended fields:
- timestamp
- process
- job name
- document/invoice identifier when safe
- batch identifier
- operation
- duration
- status
- error category
- correlation ID

Never log passwords, private keys, access tokens, or complete sensitive payloads.

## Environment configuration

Use environment variables or deployment configuration.

Never commit:
- database passwords
- SMTP passwords
- API tokens
- certificates/private keys
- production connection strings containing secrets

Validate required configuration during startup.

## Diagnostics

When a Node.js process is slow or unstable, inspect:
1. CPU
2. memory/RSS
3. event-loop blocking
4. database pool usage
5. active database requests
6. external HTTP latency
7. file I/O
8. batch size
9. retry frequency
10. PM2 restarts

Do not assume high CPU means Node.js itself is the root cause; PDF rendering, compression, cryptography, JSON processing, or child processes may be responsible.

## Version awareness

Node.js behavior differs across versions, especially ESM/CommonJS and package compatibility.

When `ERR_REQUIRE_ESM` appears, inspect Node.js version, package version, package `type`, import/require usage, and dependency constraints before recommending a downgrade.

## Output style

Default:
### Diagnosis
State whether the cause is confirmed or suspected.

### Recommended change
Give the smallest practical change.

### Code
Only relevant code unless the user asks for a full project.

### Validation
Explain how to prove the change worked.

For production changes add:
### Risk
State concurrency, data integrity, compatibility, and deployment risks.

## Reference selection

Load only relevant references:
- `references/architecture.md`
- `references/async-concurrency.md`
- `references/database-integration.md`
- `references/batch-workers.md`
- `references/pm2-windows.md`
- `references/http-retries.md`
- `references/logging.md`
- `references/pdf-email.md`
- `references/xml-sifen.md`
- `references/diagnostics.md`
