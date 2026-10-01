# Database Integration

Use connection pools for SQL Server, DB2, Oracle, and similar databases.

Check pool size, acquire timeout, connection timeout, query timeout, connection release, and retry behavior.

Do not increase pool size without checking database capacity and concurrency.

If SQL Server is the bottleneck, delegate locking, waits, indexes, transactions, and execution plans to `sqlserver`.
