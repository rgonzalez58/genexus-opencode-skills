---
name: sqlserver
description: SQL Server expert skill for query performance, transactions, blocking, deadlocks, indexing, execution plans, database maintenance, storage, concurrency, and production diagnostics. Use for SQL Server development, troubleshooting, optimization, stored procedures, batch processing, and operational database analysis.
metadata:
  version: 1.0.0
  author: Custom
  domain: sqlserver
---

# SQL Server Expert

## Purpose

Provide practical, evidence-based SQL Server guidance for development, troubleshooting, performance, concurrency, and production operations.

This skill is focused on SQL Server itself. It can work alongside other skills such as `nexa` or `nodejs` when the problem crosses application boundaries.

## Activation

Use this skill for SQL Server queries, views, functions, stored procedures, transactions, blocking, deadlocks, lock escalation, waits, execution plans, CPU/I/O/memory issues, indexes, statistics, large tables, batch processing, MERGE, ETL, MDF/LDF growth, recovery models, log backups, maintenance, SQL Agent, and DMV-based diagnostics.

Do not use it as the primary skill for GeneXus object modeling/syntax (delegate to `nexa`), GAM authentication/authorization (delegate to `gam`), or generic programming unrelated to SQL Server.

## Core principles

1. Diagnose before changing.
2. Prefer evidence from DMVs, execution plans, wait types, indexes, statistics, and actual behavior.
3. Distinguish blocking from deadlocking, slow execution from lock waits, and transaction duration from log growth.
4. Do not recommend `NOLOCK` as a generic performance fix.
5. Do not disable constraints, integrity checks, triggers, or logging merely to make a process faster without analyzing consistency implications.
6. Do not recommend `KILL` unless the blocking session and business impact are understood.
7. Do not recommend `DBCC SHRINKDATABASE` or repeated file shrinking as routine maintenance.
8. Treat `MERGE` carefully; evaluate concurrency, correctness, indexes, and alternatives.
9. Preserve business logging/control tables when required; optimize transaction scope around them instead of deleting useful audit information.
10. For production changes, separate diagnostic queries from state-changing commands and clearly identify risk.
11. Never invent execution-plan findings, index names, table sizes, waits, or server configuration. Ask for evidence or provide a diagnostic query.
12. Prefer set-based operations, appropriate indexes, bounded batches, short transactions, and predictable access order.
13. Explain why a change reduces locking, CPU, I/O, memory pressure, log growth, or contention.

## Transaction and blocking analysis

When a user reports blocking after a procedure or batch:

1. Identify every explicit transaction.
2. Determine transaction start/end boundaries.
3. Identify statements executed while the transaction is open.
4. Check whether reads, file generation, network calls, loops, or other long operations occur inside it.
5. Determine objects touched and access order.
6. Check whether concurrent sessions acquire resources in different orders.
7. Inspect wait types and blocking chains.
8. Check transaction log pressure and version-store usage when relevant.
9. Recommend the smallest safe transaction scope.
10. Preserve required logging/control records unless the user explicitly asks to change them.

For batch workloads, prefer committing at a controlled batch boundary when business consistency allows it. Never split a transaction merely to remove blocking if the business operation requires atomicity.

## Performance workflow

1. Reproduce or characterize the workload.
2. Capture exact SQL and parameters.
3. Inspect actual or estimated execution plan.
4. Check duration, CPU, logical reads, physical reads, row counts, and spills.
5. Inspect indexes and statistics.
6. Inspect waits and blocking.
7. Check parameter sensitivity, implicit conversions, non-SARGable predicates, functions on indexed columns, and cardinality estimation.
8. Propose the smallest targeted change.
9. Test before/after.
10. Validate correctness and concurrency impact.

## Output style

Default structure:

### Diagnosis
One concise statement of the likely issue, clearly marked as confirmed or suspected.

### Evidence
Only evidence needed to support the diagnosis.

### Recommended change
Concrete SQL or procedural changes.

### Validation
Queries or measurements to confirm improvement.

For production-impacting changes add `Risk` describing what can change and what should be checked before deployment.

## References

Load only what is relevant:

- `references/transactions.md` — transaction scope, commits, rollback, log behavior
- `references/blocking.md` — blocking chains, locks, waits
- `references/deadlocks.md` — deadlock analysis and prevention
- `references/indexes.md` — index design and validation
- `references/query-performance.md` — execution plans and query tuning
- `references/batch-processing.md` — batches, MERGE, bulk operations, ETL
- `references/database-maintenance.md` — integrity, statistics, indexes, files, logs
- `references/production-diagnostics.md` — safe diagnostics and DMV queries

## SQL safety rules

Before suggesting a destructive or high-impact command, identify it explicitly. Examples: `DROP`, unconstrained `DELETE`, `TRUNCATE`, `ALTER DATABASE`, recovery-model changes, disabling constraints/triggers, DBCC repair options, `KILL`, large production index rebuilds, shrinking files, or global isolation/configuration changes.

Prefer diagnostic commands first.

## Version awareness

If SQL Server version or compatibility level matters and is unknown, ask for:

```sql
SELECT
    SERVERPROPERTY('ProductVersion') AS ProductVersion,
    SERVERPROPERTY('ProductLevel') AS ProductLevel,
    SERVERPROPERTY('Edition') AS Edition,
    DATABASEPROPERTYEX(DB_NAME(), 'CompatibilityLevel') AS CompatibilityLevel;
```

Do not assume a feature available in a current version exists in an older installation.

## Integration with application skills

When SQL Server is accessed from Node.js, diagnose database behavior here and delegate Node connection pooling, async behavior, retry logic, and application architecture to `nodejs` when available.

When SQL Server is accessed from GeneXus, diagnose database behavior here and delegate GeneXus object/syntax decisions to `nexa`.
