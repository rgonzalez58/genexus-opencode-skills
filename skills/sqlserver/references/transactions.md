# Transactions and Lock Duration

Keep transactions as short as business correctness permits.

## Diagnostic questions

- Where does `BEGIN TRAN` occur?
- Is there one transaction per procedure or around an entire batch?
- Does it contain loops, cursors, large reads, external calls, file operations, or waits?
- Are commits performed at reasonable batch boundaries?
- Can errors leave an open transaction?
- Which tables are modified before commit?

## Diagnostic query

```sql
SELECT r.session_id,r.status,r.command,r.wait_type,r.wait_time,
       r.blocking_session_id,r.open_transaction_count,r.cpu_time,
       r.total_elapsed_time,r.logical_reads,r.reads,r.writes,
       DB_NAME(r.database_id) AS database_name,t.text AS sql_text
FROM sys.dm_exec_requests AS r
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE r.session_id <> @@SPID
ORDER BY r.total_elapsed_time DESC;
```

Prefer a narrow atomic scope:

```sql
BEGIN TRY
    BEGIN TRAN;
    -- only the atomic database changes
    COMMIT;
END TRY
BEGIN CATCH
    IF XACT_STATE() <> 0 ROLLBACK;
    THROW;
END CATCH;
```

Batch commits can reduce lock duration and log pressure only when each batch is independently consistent. Preserve required audit/control logging; move nonessential work outside the critical transaction where possible.
