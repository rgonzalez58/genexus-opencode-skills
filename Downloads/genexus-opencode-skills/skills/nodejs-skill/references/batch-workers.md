# Batch and Workers

A reliable worker should be restartable.

Recommended state:

```text
PENDING -> PROCESSING -> SUCCESS
                    \-> ERROR
```

Use a safe claim mechanism so two workers do not process the same record.

Bound batch size. Commit database changes at deliberate business boundaries.

Persist progress and errors so a restart can resume without reprocessing successful work.
