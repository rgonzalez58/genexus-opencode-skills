# Production Diagnostics

Safe first response:

1. Identify affected database and time window.
2. Inspect active requests.
3. Identify blockers.
4. Inspect waits.
5. Identify long-running transactions.
6. Inspect CPU/I/O/memory indicators.
7. Capture SQL and plan where possible.
8. Only then consider intervention.

```sql
SELECT SERVERPROPERTY('ProductVersion') AS ProductVersion,
       SERVERPROPERTY('ProductLevel') AS ProductLevel,
       SERVERPROPERTY('Edition') AS Edition;

SELECT DB_NAME() AS database_name,
       DATABASEPROPERTYEX(DB_NAME(),'Recovery') AS RecoveryModel,
       DATABASEPROPERTYEX(DB_NAME(),'CompatibilityLevel') AS CompatibilityLevel;
```

```sql
SELECT session_id,status,command,blocking_session_id,wait_type,wait_time,
       cpu_time,total_elapsed_time,logical_reads,reads,writes,open_transaction_count
FROM sys.dm_exec_requests
WHERE session_id <> @@SPID
ORDER BY total_elapsed_time DESC;
```

Never issue `KILL`, disable constraints, change recovery model, shrink files, or change global isolation merely because a system is slow. Establish the cause first.
