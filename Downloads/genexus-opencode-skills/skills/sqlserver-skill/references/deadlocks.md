# Deadlocks

A deadlock occurs when sessions wait on each other in a cycle. SQL Server chooses a victim and rolls back that transaction.

Prevent with consistent object access order, short transactions, appropriate indexes, bounded batches, and no external waits while holding locks.

For confirmed deadlocks, prefer Extended Events and deadlock graphs. Do not infer a deadlock solely from slow queries or blocking.

Application retries can be appropriate for transient deadlock victims, but must be bounded and the operation must be safe to repeat.
