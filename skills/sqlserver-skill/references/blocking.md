# Blocking and Locks

Blocking means one session is waiting for another to release a resource. It is not automatically a deadlock.

## Current blocking chain

```sql
SELECT r.session_id,r.blocking_session_id,r.status,r.wait_type,r.wait_time,
       r.wait_resource,r.command,DB_NAME(r.database_id) AS database_name,
       s.host_name,s.program_name,s.login_name,t.text AS sql_text
FROM sys.dm_exec_requests AS r
LEFT JOIN sys.dm_exec_sessions AS s ON s.session_id=r.session_id
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE r.blocking_session_id <> 0
ORDER BY r.wait_time DESC;
```

## Lock inspection

```sql
SELECT request_session_id,resource_type,resource_database_id,
       resource_associated_entity_id,request_mode,request_status
FROM sys.dm_tran_locks
ORDER BY request_session_id,resource_type;
```

Typical causes: long transactions, inefficient indexes, large updates/deletes, inconsistent object order, lock escalation, high-concurrency updates, and uncommitted transactions.

Remediation order: find the head blocker; identify its SQL and transaction; determine why it holds the lock; reduce transaction duration/work; improve access paths; standardize object order; consider row-versioning only after understanding implications.

Do not use `NOLOCK` as a generic solution.
