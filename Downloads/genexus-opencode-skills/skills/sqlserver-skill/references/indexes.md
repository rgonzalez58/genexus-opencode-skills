# Indexes

An index should support an actual access pattern. Evaluate equality/range predicates, JOINs, ORDER BY, GROUP BY, selectivity, included columns, write overhead, and overlapping indexes.

```sql
SELECT i.name AS index_name,i.type_desc,i.is_unique,i.is_primary_key,
       c.name AS column_name,ic.key_ordinal,ic.is_included_column
FROM sys.indexes AS i
JOIN sys.index_columns AS ic ON ic.object_id=i.object_id AND ic.index_id=i.index_id
JOIN sys.columns AS c ON c.object_id=ic.object_id AND c.column_id=ic.column_id
WHERE i.object_id=OBJECT_ID(N'dbo.TuTabla')
ORDER BY i.index_id,ic.key_ordinal,ic.is_included_column;
```

Do not recommend an index without considering write cost and existing indexes. Fragmentation alone is not sufficient reason for a rebuild; consider workload, page count, page density, and concurrency.
