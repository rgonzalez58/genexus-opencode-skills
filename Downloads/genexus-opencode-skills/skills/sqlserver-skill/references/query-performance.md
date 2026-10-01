# Query Performance

Collect exact SQL, parameters, duration, CPU, logical reads, actual execution plan when possible, row counts, waits, and version/compatibility level.

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
-- query
SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
```

Common causes include unsuitable indexes, non-SARGable predicates, implicit conversions, functions on indexed columns, poor cardinality estimates, parameter sensitivity, spills, key lookups, large memory grants, and blocking mistaken for execution time.

Do not optimize from estimated cost percentages alone. Validate with runtime metrics.
