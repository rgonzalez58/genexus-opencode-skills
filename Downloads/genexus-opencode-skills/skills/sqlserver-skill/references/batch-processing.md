# Batch Processing and ETL

Define batch size, ordering key, retry behavior, transaction boundary, progress/control logging, failure recovery, and idempotency.

Prefer deterministic TOP(N) or stable key ranges over repeatedly scanning an entire table.

For `MERGE`, review matching uniqueness, source duplicates, indexes, concurrency, transaction scope, error handling, version, and alternatives. Do not assume `MERGE` is automatically faster or safer.

Preserve control/audit tables when they are required for reconciliation or control files. Optimize when/how they are written rather than deleting them.

For ETL separate extraction, staging, transformation, target writes, and finalization/logging, and measure each phase independently.
