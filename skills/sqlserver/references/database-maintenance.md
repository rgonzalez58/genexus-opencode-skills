# Database Maintenance

Use `DBCC CHECKDB` for integrity validation. Repair options are exceptional and require a recovery plan.

Evaluate statistics by modification volume, cardinality, and query behavior. Do not rebuild every index on a schedule without evidence.

Analyze log growth through recovery model, log-backup frequency, long transactions, large modifications, and other workload features. Do not use shrinking as routine log maintenance; remove the cause first and establish an appropriate file size.

```sql
SELECT DB_NAME(database_id) AS database_name,type_desc,name,physical_name,
       size*8.0/1024 AS size_mb,growth,is_percent_growth
FROM sys.master_files
ORDER BY database_id,type_desc,name;
```
